---
description: "Task list for host metrics integration"
---

# Tasks: Host Metrics Integration

**Input**: Design documents from `specs/002-host-metrics/`
**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, data-model.md ✅, contracts/playbook-interface.md ✅

**Tests**: Not requested — ansible-lint validation included in Polish phase.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2)
- Include exact file paths in all descriptions

## Path Conventions

- Playbook: `playbooks/count_managed_nodes.yml`
- Documentation: `docs/`
- Specs/design: `specs/002-host-metrics/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Confirm existing playbook structure before extending it

- [x] T001 Review `playbooks/count_managed_nodes.yml` and confirm `api_base`, `host_metrics_available`, and `count_result` facts are understood — no file changes; this is a read-only checkpoint before any edits begin

**Checkpoint**: Existing playbook structure confirmed — ready to add metrics tasks

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Add the metrics probe and default fact that all US1/US2 tasks depend on

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [x] T002 Add "Probe host metrics endpoint" task to `playbooks/count_managed_nodes.yml` immediately after the "Confirm CSV written" task (after existing T015): use `ansible.builtin.uri` with `url: "{{ api_base }}/host_metrics/?page_size=1"`, auth headers matching the existing pattern, `validate_certs: false`, `status_code: [200, 404, 403, 500]`, `failed_when: false`, `changed_when: false`; register result as `metrics_probe`

- [x] T003 Add "Set host metrics availability and defaults" task to `playbooks/count_managed_nodes.yml` immediately after T002: use `ansible.builtin.set_fact` to set `host_metrics_available: "{{ metrics_probe.status == 200 }}"` and `host_metrics_summary` dict with default/empty values for all five fields (`total_in_metrics: 0`, `automated: 0`, `not_automated: 0`, `most_recent_automation: ''`, `available: false`); also add a `ansible.builtin.debug` task to log the appropriate message for 404 ("Not available on this platform"), 403 ("Not available — permission denied"), or unexpected status codes

**Checkpoint**: Probe and defaults in place — US1 and US2 implementation can now begin

---

## Phase 3: User Story 1 - Host Metrics in Job Log Report (Priority: P1) 🎯 MVP

**Goal**: Job log report includes a populated Host Metrics section when the endpoint is available; prints a graceful "not available" message on Tower or when permissions are insufficient

**Independent Test**: Launch the job template against AAP 2.x — verify the job log shows the Host Metrics section with `Hosts in Metrics`, `Automated (≥1 run)`, `Never Automated`, and `Most Recent Run` values populated. Then launch against an older instance (or simulate 404 by pointing to Tower) — verify job succeeds with "Not available on this platform" message.

### Implementation for User Story 1

- [x] T004 [US1] Add "Fetch automated host count" task to `playbooks/count_managed_nodes.yml` after T003: use `ansible.builtin.uri` with `url: "{{ api_base }}/host_metrics/?automated_counter__gt=0&page_size=1"`, same auth headers, `validate_certs: false`, `status_code: [200, 400]`, `failed_when: false`, `changed_when: false`; register as `metrics_automated`; guard with `when: host_metrics_available`

- [x] T005 [US1] Add "Fetch most recent automation date" task to `playbooks/count_managed_nodes.yml` after T004: use `ansible.builtin.uri` with `url: "{{ api_base }}/host_metrics/?ordering=-last_automation&page_size=1"`, same auth/validate pattern, `status_code: [200]`, `failed_when: false`, `changed_when: false`; register as `metrics_recent`; guard with `when: host_metrics_available`

- [x] T006 [US1] Add "Build host_metrics_summary" task to `playbooks/count_managed_nodes.yml` after T005: use `ansible.builtin.set_fact` to overwrite `host_metrics_summary` with `available: true`, `total_in_metrics: "{{ metrics_probe.json.count }}"`, `automated: "{{ metrics_automated.json.count | default(0) }}"`, `not_automated: "{{ metrics_probe.json.count | int - metrics_automated.json.count | default(0) | int }}"`, `most_recent_automation: "{{ metrics_recent.json.results[0].last_automation | default('') }}"` ; guard with `when: host_metrics_available`

- [x] T007 [US1] Update the "Build count result" `ansible.builtin.set_fact` task in `playbooks/count_managed_nodes.yml` to include five new fields in the `count_result` dict: `metrics_available`, `hosts_in_metrics`, `hosts_automated`, `hosts_not_automated`, `most_recent_automation` — sourced from `host_metrics_summary`

- [x] T008 [US1] Update the "Print managed node count report" `ansible.builtin.debug` task in `playbooks/count_managed_nodes.yml` to append a Host Metrics section after the disabled count line, matching the contract in `specs/002-host-metrics/contracts/playbook-interface.md`: show populated values when `count_result.metrics_available` is true, or show "Not available on this platform" / "Not available (permission denied)" based on `metrics_probe.status`

**Checkpoint**: User Story 1 complete — launch job and verify metrics section in job log

---

## Phase 4: User Story 2 - Host Metrics in CSV Output (Priority: P2)

**Goal**: CSV files pushed to GitHub include five new metrics columns; columns are present but empty when metrics are unavailable

**Independent Test**: After a successful run against AAP 2.x, pull the CSV from `output/` in the GitHub repo and confirm it has 11 columns with the header `timestamp,controller_url,total_count,enabled_count,disabled_count,job_id,metrics_available,hosts_in_metrics,hosts_automated,hosts_not_automated,most_recent_automation`. On a run where metrics are unavailable, confirm the last five column values are empty strings.

### Implementation for User Story 2

- [x] T009 [US2] Update the "Build CSV content for GitHub" `ansible.builtin.set_fact` task in `playbooks/count_managed_nodes.yml` to extend `csv_content_raw` with the five new columns appended after `job_id`: `{{ count_result.metrics_available }},{{ count_result.hosts_in_metrics }},{{ count_result.hosts_automated }},{{ count_result.hosts_not_automated }},{{ count_result.most_recent_automation }}`

- [x] T010 [US2] Update the "Write CSV report" `ansible.builtin.copy` task in `playbooks/count_managed_nodes.yml` to extend both the header row and data row with the five new columns in the same order as T009

- [x] T011 [US2] Update the "Publish count result as AAP artifacts" `ansible.builtin.set_stats` task in `playbooks/count_managed_nodes.yml` to include the five new metrics fields: `managed_node_count_metrics_available`, `managed_node_count_hosts_in_metrics`, `managed_node_count_hosts_automated`, `managed_node_count_hosts_not_automated`, `managed_node_count_most_recent_automation`

**Checkpoint**: User Stories 1 and 2 complete — job log shows metrics AND CSV has 11 columns in GitHub

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: Documentation updates, linting, and end-to-end validation

- [x] T012 [P] Update `docs/README.md` to add a "Host Metrics" section explaining the new metrics data in the job log and CSV, noting it requires AAP 2.4+ and gracefully degrades on Tower

- [x] T013 [P] Update `docs/limitations.md` to add a caveat that host metrics field names (`automated_counter`, `first_automation`) are inferred from documentation and may vary — advise checking the OpenAPI schema at `/api/gateway/v1/docs/schema/` if metrics show unexpected values

- [x] T014 Run `ansible-lint playbooks/count_managed_nodes.yml` from repository root and fix all reported issues (profile: production)

- [x] T015 Run quickstart.md validation scenarios end-to-end against a live AAP 2.x instance: verify metrics section in job log (Scenario 1), graceful degradation (Scenario 2), and 11-column CSV in GitHub (verification checklist)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — read-only review, start immediately
- **Foundational (Phase 2)**: Depends on Phase 1 completion; T002 → T003 (sequential — T003 uses `metrics_probe` registered in T002)
- **User Story 1 (Phase 3)**: Depends on Phase 2; T004 → T005 → T006 → T007 → T008 (sequential — each builds on prior fact)
- **User Story 2 (Phase 4)**: Depends on Phase 3 completion (needs updated `count_result` from T007)
- **Polish (Phase 5)**: T012 and T013 can run after Phase 3; T014 requires all playbook tasks complete; T015 requires T014 pass

### User Story Dependencies

- **US1 (P1)**: Depends only on Foundational (Phase 2)
- **US2 (P2)**: Depends on US1 (needs `count_result` updated with metrics fields in T007)

---

## Parallel Execution Examples

### Phase 5 Parallels (documentation)
```
Task: "Update docs/README.md with host metrics section"    → T012
Task: "Update docs/limitations.md with field name caveat"  → T013 (parallel with T012)
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (review)
2. Complete Phase 2: Foundational (probe + defaults — BLOCKS both user stories)
3. Complete Phase 3: User Story 1 (metrics in job log)
4. **STOP and VALIDATE**: Launch job in AAP GUI; confirm metrics section in job log
5. Deploy/demo if ready

### Incremental Delivery

1. Setup + Foundational → probe and defaults wired up
2. User Story 1 → metrics visible in job log (MVP demo-able)
3. User Story 2 → metrics in CSV and GitHub output
4. Polish → documentation updated, lint clean, end-to-end validated

---

## Notes

- All changes are in-place edits to `playbooks/count_managed_nodes.yml` — no new files
- [P] tasks in Phase 5 are different documentation files with no shared state
- US2 depends on US1 (T007 must set the new `count_result` fields before CSV tasks can use them)
- `failed_when: false` on the probe task (T002) ensures any unexpected HTTP status does not abort the playbook
- Field access from `metrics_probe.json`, `metrics_automated.json`, `metrics_recent.json` should use `| default(0)` / `| default('')` guards for robustness across AAP versions
