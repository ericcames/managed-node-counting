# Research: Host Metrics Integration

**Branch**: `002-host-metrics` | **Date**: 2026-04-17
**Input**: Feature spec + AAP API documentation + AWX source research

---

## Decision 1: host_metrics Endpoint Shape and Path

**Decision**: Use `GET /api/controller/v2/host_metrics/` (AAP 2.x) or
`/api/v2/host_metrics/` (Tower/AAP 1.x fallback via `api_base` established in
feature 001). The endpoint returns **per-host records** in standard AAP paginated
format (`count`, `next`, `previous`, `results`). It is NOT an aggregate summary
endpoint.

**Confirmed fields** (from Red Hat documentation and AWX source):
- `last_automation` — timestamp of most recent automation run for this host
- `first_automation` — timestamp of first automation run (inferred, consistent with
  Red Hat 2.4 release notes referencing "first seen" tracking)
- `automated_counter` — integer count of automation runs (inferred field name; Red Hat
  2.4 release notes confirm per-host run count tracking)
- `used_in_inventories` — integer count of inventories the host appears in
- `deleted` — boolean soft-deletion flag

**Important caveat**: Exact serializer field names (`automated_counter`,
`first_automation`) are inferred from documentation. The OpenAPI schema at
`/api/gateway/v1/docs/schema/?format=json` on a live instance is authoritative.
The implementation should use `| default(0, true)` or `| default('', true)` on
any field access to guard against field name differences across versions.

**Alternatives considered**:
- Aggregate summary endpoint: Does not exist — per-host records only.
- `awx-manage host_metric --csv` CLI: Requires shell access to controller — not
  available from a job execution context.

---

## Decision 2: Aggregation Strategy

**Decision**: Use two lightweight API calls rather than paginating all records:
1. `GET /api/controller/v2/host_metrics/?page_size=1` → read `.count` for total
   hosts tracked in the metrics system.
2. `GET /api/controller/v2/host_metrics/?automated_counter__gt=0&page_size=1` →
   read `.count` for hosts automated at least once. If this filter param is not
   supported (returns all records), fall back to: `total - never_automated`.
3. `GET /api/controller/v2/host_metrics/?ordering=-last_automation&page_size=1` →
   read `results[0].last_automation` for the most recent automation date.

This gives accurate aggregate counts with O(1) API calls regardless of fleet size,
satisfying SC-004 (≤5 second overhead).

**Alternatives considered**:
- Full pagination + client-side aggregation: Accurate but O(n) API calls for large
  fleets — violates SC-004 for large environments. Rejected.
- Single page_size=200 fetch: Accurate only for fleets ≤200 hosts. Rejected as
  incorrect for large deployments.

---

## Decision 3: Graceful Degradation Implementation

**Decision**: Use `ansible.builtin.uri` with `status_code: [200, 404, 403]` and
`ignore_errors: false` (letting the allowed status codes handle non-failures). Set a
boolean fact `host_metrics_available` based on the registered result's `.status`.
All subsequent metrics tasks use `when: host_metrics_available` conditionals.

Status handling:
- `200` → metrics available, process data
- `404` → endpoint absent (Tower/AAP 1.x), set `host_metrics_available: false`,
  log informational message
- `403` → insufficient permissions, set `host_metrics_available: false`,
  log warning with permission guidance
- Any other status (e.g., 500) → captured via `failed_when: false` on a broader
  `status_code: [200, 404, 403, 500]` list; log unexpected status as warning

**Alternatives considered**:
- `ignore_errors: true` on the URI task: Would suppress all errors including
  connectivity failures — too broad. Rejected.
- Separate `ansible.builtin.assert` for metrics: Would fail the job on missing
  endpoint — violates FR-004. Rejected.

---

## Decision 4: CSV Schema Extension

**Decision**: Append new columns to the right of the existing CSV schema. Existing
column order is preserved for backward compatibility:

```
timestamp,controller_url,total_count,enabled_count,disabled_count,job_id,
metrics_available,hosts_in_metrics,hosts_automated,hosts_not_automated,
most_recent_automation
```

When `host_metrics_available` is false, the metrics columns are written as empty
strings. This satisfies FR-007 (consistent schema) and SC-003 (consistent columns
across all runs).

**Alternatives considered**:
- Separate metrics CSV file: Would require a second GitHub push and two credential
  calls — adds complexity without benefit. Rejected (Principle V).
- Omit metrics columns when unavailable: Breaks CSV schema consistency, making
  automated parsing fragile. Rejected.

---

## Decision 5: No Credential Changes Required

**Decision**: The existing `CONTROLLER_OAUTH_TOKEN` / `CONTROLLER_USERNAME` +
`CONTROLLER_PASSWORD` credential provides sufficient access to the host_metrics
endpoint under normal AAP permissions. No new credential type fields are needed.

**Rationale**: The host_metrics endpoint is a read-only GET endpoint accessible to
any user with the System Auditor role — the same minimum permission required for the
existing `/hosts/` endpoint.

**Alternatives considered**:
- New credential field for metrics token: Unnecessary complexity. Rejected.
- Separate credential type for metrics: Over-engineered for a single endpoint
  reusing the same auth. Rejected (Principle V).
