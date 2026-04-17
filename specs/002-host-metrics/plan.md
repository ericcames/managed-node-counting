# Implementation Plan: Host Metrics Integration

**Branch**: `002-host-metrics` | **Date**: 2026-04-17 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `specs/002-host-metrics/spec.md`

## Summary

Extend `playbooks/count_managed_nodes.yml` to query the AAP 2.x `host_metrics`
endpoint after the existing node count logic. Use `ansible.builtin.uri` with a
permissive `status_code` list to probe availability, set a `host_metrics_available`
boolean fact, and conditionally process metrics data. Append five new fields to the
existing CSV row and extend the job log report block. No new credentials or job
template changes are required.

## Technical Context

**Language/Version**: YAML (Ansible playbook), ansible-core 2.15 minimum (AAP 2.4 baseline)
**Primary Dependencies**: `ansible.builtin` only — `uri`, `set_fact`, `debug`, `copy` (no additional collections)
**Storage**: GitHub repository `output/` directory via Contents API (same as feature 001)
**Testing**: ansible-lint (profile: production); manual end-to-end validation on live AAP instance
**Target Platform**: AAP 2.4, 2.5, 2.6 (full support); Ansible Tower (graceful degradation)
**Project Type**: Ansible playbook extending an existing playbook
**Performance Goals**: Metrics retrieval adds ≤5 seconds to total job run time (SC-004)
**Constraints**: No extra vars; no additional collections; `ansible.builtin` only; backward-compatible CSV schema
**Scale/Scope**: Same fleet sizes as feature 001; metrics endpoint called with `page_size=1` (O(1) API calls)

## Constitution Check

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Accuracy First | ✅ PASS | Metrics counts sourced directly from API `.count` fields; no estimation |
| II. Idempotent Operations | ✅ PASS | Read-only API calls; CSV overwrite behavior unchanged; no state mutation |
| III. Test-First | ✅ PASS | Quickstart scenarios defined; ansible-lint validates before deploy |
| IV. Observability | ✅ PASS | All three degradation paths (404, 403, 5xx) log explicit messages; no silent failures |
| V. Simplicity | ✅ PASS | Two additional API calls (page_size=1); no new collections; no new credential fields |

No violations to justify.

## Project Structure

### Documentation (this feature)

```text
specs/002-host-metrics/
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
├── README.md                        # Updated: host metrics section added
└── limitations.md                   # Updated: metrics field name caveat added

output/                              # Runtime — CSV files pushed to GitHub
```

**Structure Decision**: Single-file playbook extension. All changes are in-place
modifications to `playbooks/count_managed_nodes.yml`. No new source files created
(Principle V).

## Implementation Approach

### New tasks to add (in order, after existing T015 "Confirm CSV written"):

1. **Probe host metrics endpoint** — `ansible.builtin.uri` with
   `url: "{{ api_base }}/host_metrics/?page_size=1"`,
   `status_code: [200, 404, 403, 500]`, `failed_when: false`.
   Register as `metrics_probe`.

2. **Set host_metrics_available fact** — `ansible.builtin.set_fact` based on
   `metrics_probe.status == 200`. Also set `host_metrics_summary` with zero/empty
   defaults for all fields.

3. **Fetch automated host count** (when available) — `ansible.builtin.uri` with
   `url: "{{ api_base }}/host_metrics/?automated_counter__gt=0&page_size=1"`.
   Register as `metrics_automated`. Guard with `when: host_metrics_available`.

4. **Fetch most recent automation date** (when available) — `ansible.builtin.uri`
   with `url: "{{ api_base }}/host_metrics/?ordering=-last_automation&page_size=1"`.
   Register as `metrics_recent`. Guard with `when: host_metrics_available`.

5. **Build host_metrics_summary fact** — `ansible.builtin.set_fact` to assemble
   the final dict from the probe count, automated count, and most recent date.
   Guard with `when: host_metrics_available`.

6. **Extend count_result fact** — update existing `count_result` to include all
   five new fields from `host_metrics_summary`.

7. **Print metrics in job log** — Extend the existing debug task (T013) to include
   the host metrics section. Use `when`-based conditional lines or a single task
   that always runs with conditional content.

8. **Update CSV write task** — Extend header and data row in the `ansible.builtin.copy`
   task to include the five new columns.

9. **Update CSV content for GitHub push** — Extend `csv_content_raw` fact to include
   the five new columns.

10. **Update set_stats** — Add metrics fields to the `ansible.builtin.set_stats`
    artifacts block.

### Graceful degradation logic

```
metrics_probe.status == 200  → process metrics (tasks 3–5 execute)
metrics_probe.status == 404  → log "Not available on this platform", skip tasks 3–5
metrics_probe.status == 403  → log "Permission denied", skip tasks 3–5
metrics_probe.status == other → log "Unexpected status {{ metrics_probe.status }}", skip tasks 3–5
```

All paths write the five CSV columns (empty string when unavailable).

## Complexity Tracking

No constitution violations. No complexity exceptions required.
