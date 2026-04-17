# Implementation Plan: Job Events Per-Host Enrichment

**Branch**: `003-job-events-hosts` | **Date**: 2026-04-17 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `specs/003-job-events-hosts/spec.md`

## Summary

Extend `playbooks/count_managed_nodes.yml` to query the AAP `job_events`
endpoint across the last 10 completed jobs. Aggregate results to a per-host
last-status summary (ok / failed / unreachable). Print a "Recent Host
Activity" section in the job log and append two aggregate count columns to
the CSV. All changes are in-place extensions to the existing playbook using
`ansible.builtin` only. No new credentials or job template changes required.

## Technical Context

**Language/Version**: YAML (Ansible playbook), ansible-core 2.15 minimum (AAP 2.4 baseline)
**Primary Dependencies**: `ansible.builtin` only — `uri`, `set_fact`, `debug`, `copy` (no additional collections)
**Storage**: GitHub repository `output/` directory via Contents API (unchanged)
**Testing**: ansible-lint (profile: production); manual end-to-end validation on live AAP instance
**Target Platform**: AAP 2.4, 2.5, 2.6; Ansible Tower (graceful degradation)
**Performance Goals**: Event retrieval adds ≤10 seconds to total job run time (SC-004)
**Constraints**: No extra vars; no additional collections; backward-compatible 13-column CSV; bounded to last 10 jobs
**Scale/Scope**: Assumes ≤200 host-level events across 10 jobs (single page); pagination loop handles larger fleets

## Constitution Check

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Accuracy First | ✅ PASS | Last-status per host derived from actual event records; paginated to capture all events |
| II. Idempotent Operations | ✅ PASS | Read-only API calls; CSV overwrite unchanged; no state mutation |
| III. Test-First | ✅ PASS | Quickstart scenarios defined; ansible-lint validates before deploy |
| IV. Observability | ✅ PASS | All degradation paths log explicit messages; no silent failures |
| V. Simplicity | ✅ PASS | 2 additional API calls; no new collections; no new credentials |

No violations to justify.

## Project Structure

### Documentation (this feature)

```text
specs/003-job-events-hosts/
├── plan.md                          # This file
├── research.md                      # Phase 0 output
├── data-model.md                    # Phase 1 output
├── quickstart.md                    # Phase 1 output
├── contracts/
│   └── playbook-interface.md        # Phase 1 output
└── tasks.md                         # Phase 2 output (speckit-tasks)
```

### Source Code (repository root)

```text
playbooks/
└── count_managed_nodes.yml          # Extended in-place (no new files)

docs/
├── README.md                        # Updated: job events section added
└── limitations.md                   # Updated: event history caveat added
```

**Structure Decision**: Single-file playbook extension. All changes in-place
in `playbooks/count_managed_nodes.yml`. No new source files (Principle V).

## Implementation Approach

New tasks insert after the "Build host metrics summary" block and before
"Build count result". This ensures all enrichment data is available when
`count_result` is assembled.

### New tasks (in order):

1. **Fetch last 10 completed job IDs** — `ansible.builtin.uri`
   `GET {{ api_base }}/jobs/?status__in=successful,failed&order_by=-id&page_size=10`
   Register as `recent_jobs`. Set `failed_when: false`.

2. **Build recent job IDs list** — `ansible.builtin.set_fact`
   Extract IDs, exclude current `job_id`, join as comma-separated string.
   Set `job_events_available: false` and empty defaults if no jobs found.

3. **Fetch job events** (when IDs available) — `ansible.builtin.uri`
   `GET {{ api_base }}/job_events/?job__in=<ids>&host_name__gt=&event__startswith=runner_on&page_size=200`
   Register as `job_events_page1`.

4. **Fetch remaining event pages** (when `next` is not null) — loop.

5. **Accumulate all event records** — `ansible.builtin.set_fact` `all_job_events`.

6. **Get unique hostnames** — `ansible.builtin.set_fact`
   `| map(attribute='host_name') | map('lower') | select('ne', '') | unique | list`

7. **Build per-host last-status list** — `ansible.builtin.set_fact` with loop
   For each hostname: most recent event → map to ok/failed/unreachable/skipped.

8. **Build job_event_aggregates** — count failures and successes.

9. **Update count_result** — add `job_events_available`, `hosts_with_recent_failures`,
   `hosts_with_recent_success`.

10. **Update Print report** — append "Recent Host Activity" section.

11. **Update CSV tasks** — append 2 new columns to local write and GitHub push.

12. **Update set_stats** — add 3 new artifact fields.

### Event type → status mapping

| event value              | status       | Counts as failure |
|--------------------------|--------------|-------------------|
| `runner_on_ok`           | ok           | No                |
| `runner_on_changed`      | ok           | No                |
| `runner_on_skipped`      | skipped      | No                |
| `runner_on_failed`       | failed       | Yes               |
| `runner_on_unreachable`  | unreachable  | Yes               |
| other                    | unknown      | No                |

## Complexity Tracking

No constitution violations. No complexity exceptions required.
