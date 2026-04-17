# Research: Job Events Per-Host Enrichment

**Branch**: `003-job-events-hosts` | **Date**: 2026-04-17
**Input**: Feature spec + AWX source research + AAP API documentation

---

## Decision 1: job_events Endpoint Path and Query Strategy

**Decision**: Use the global endpoint `GET {{ api_base }}/job_events/` with
`?job__in=ID1,ID2,...` filter rather than per-job `/jobs/{id}/job_events/`.
This allows fetching all relevant events in one (or few) API calls instead of
one call per job.

**Confirmed fields** (from AWX source `awx/main/models/events.py`):
- `event` — event type string: `runner_on_ok`, `runner_on_failed`,
  `runner_on_unreachable`, `runner_on_skipped`, `playbook_on_*`
- `host_name` — hostname as string (max 1024 chars); empty for playbook-level events
- `created` — ISO 8601 timestamp of the event
- `task` — task name
- `job` — integer job ID
- `failed` — boolean
- `changed` — boolean

**Filter approach**:
1. `GET {{ api_base }}/jobs/?status__in=successful,failed&order_by=-id&page_size=10`
   → get the 10 most recent completed job IDs (excluding current job)
2. `GET {{ api_base }}/job_events/?job__in=<ids>&event__startswith=runner_on&host_name__gt=&page_size=200`
   → fetch host-level events only (non-empty host_name, runner_on_* events)
   Filter `host_name__gt=` (greater than empty string) excludes playbook-level events.

**Alternatives considered**:
- Per-job fetch loop: 10 API calls instead of 1 — rejected (Principle V, SC-004).
- Fetching all event types then filtering client-side: Returns large volumes of
  playbook_on_* events irrelevant to per-host status — rejected.

---

## Decision 2: Per-Host Last-Status Aggregation in Ansible

**Decision**: Use a two-step `ansible.builtin.set_fact` approach:
1. Extract unique hostnames: `| map(attribute='host_name') | unique | list`
2. Loop over hostnames, building a list of per-host dicts with:
   - `hostname`: the host (lowercase-normalized)
   - `last_status`: mapped from event type (`runner_on_ok`/`runner_on_changed` →
     `ok`, `runner_on_failed` → `failed`, `runner_on_unreachable` → `unreachable`)
   - `last_seen`: timestamp from most recent event for that host

**Event type → status mapping**:
```
runner_on_ok        → ok
runner_on_changed   → ok (changed counts as successful)
runner_on_skipped   → skipped (neutral)
runner_on_failed    → failed
runner_on_unreachable → unreachable
```
Only `failed` and `unreachable` increment `hosts_with_recent_failures`.

**Alternatives considered**:
- Server-side aggregation (not available in AAP API — per-record only).
- Jinja2 `groupby` filter: Returns tuples, harder to manipulate into dicts in
  Ansible — rejected in favour of explicit loop.

---

## Decision 3: Bounding the Query

**Decision**: Fetch the last 10 completed jobs (status `successful` or `failed`,
ordered by `-id`, `page_size=10`). Exclude the current job ID (from `job_id`
fact) if it appears in the results. This guarantees a bounded, deterministic
query with at most ~2000 events (200 hosts × 10 jobs), well within a single
200-record page for most fleets.

**Alternatives considered**:
- Time-based window (last 30 days): Non-deterministic runtime on active systems
  — rejected.
- Last 5 jobs: Too few for meaningful trend — rejected.
- Last 20 jobs: May produce 4000+ events requiring pagination — acceptable but
  conservative default of 10 chosen for SC-004 compliance.

---

## Decision 4: Pagination Handling for Job Events

**Decision**: Use `page_size=200` for the events fetch. If `next` is not null,
fetch remaining pages (same loop pattern as feature 001's host pagination).
In practice, 10 jobs × average 20 hosts × average 5 tasks = ~1000 events,
fitting in a single page. The pagination loop handles edge cases without
impacting the common path.

**Alternatives considered**:
- Hard limit at one page (200 events): Silently drops data on larger runs —
  rejected as violating Principle I (Accuracy First).

---

## Decision 5: CSV Schema Extension

**Decision**: Append 2 columns to the existing 13-column schema (11 from
features 001+002):

```
...,hosts_with_recent_failures,hosts_with_recent_success
```

When event data is unavailable, both columns are empty strings.

**Rationale**: Per-host detail rows are not added (CSV stays one row per run).
Aggregate counts give managers a health trend metric without changing the
schema cardinality.
