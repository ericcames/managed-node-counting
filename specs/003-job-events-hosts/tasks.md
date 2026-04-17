---
description: "Task list for job events per-host enrichment"
---

# Tasks: Job Events Per-Host Enrichment

**Input**: Design documents from `specs/003-job-events-hosts/`
**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, data-model.md ✅, contracts/playbook-interface.md ✅

**Tests**: Not requested — ansible-lint validation included in Polish phase.

**Organization**: Tasks grouped by user story to enable independent implementation and testing.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2)
- Include exact file paths in all descriptions

## Path Conventions

- Playbook: `playbooks/count_managed_nodes.yml`
- Documentation: `docs/`
- Specs/design: `specs/003-job-events-hosts/`

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Confirm existing playbook structure before extending it

- [x] T001 Review `playbooks/count_managed_nodes.yml` and confirm the insertion point — new tasks go after "Build host metrics summary" (the last `when: host_metrics_available` task) and before "Build count result"; confirm `job_id`, `api_base`, `controller_token`, `controller_user`, `controller_pass` facts are all in scope at that point

**Checkpoint**: Insertion point confirmed — ready to add job events tasks

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Fetch the last 10 completed job IDs and set up defaults; all US1/US2 tasks depend on this

**⚠️ CRITICAL**: No user story work can begin until this phase is complete

- [x] T002 Add "Fetch last 10 completed jobs" task to `playbooks/count_managed_nodes.yml` at the insertion point from T001: use `ansible.builtin.uri` with `url: "{{ api_base }}/jobs/?status__in=successful,failed&order_by=-id&page_size=10"`, auth headers matching the existing pattern, `validate_certs: false`, `status_code: [200]`, `failed_when: false`, `changed_when: false`; register as `recent_jobs`

- [x] T003 Add "Build recent job IDs list" task to `playbooks/count_managed_nodes.yml` immediately after T002: use `ansible.builtin.set_fact` to build `recent_job_ids` as the list of IDs from `recent_jobs.json.results | default([]) | map(attribute='id') | map('string') | select('ne', job_id | string) | list`; set `recent_job_ids_str` as `recent_job_ids | join(',')` and `job_events_available: false` with default aggregates `hosts_with_recent_failures: ''` and `hosts_with_recent_success: ''`; also add a `ansible.builtin.debug` task to log how many recent jobs were found or "No completed jobs found for event query" if the list is empty

**Checkpoint**: Job IDs list and defaults in place — US1 and US2 implementation can begin

---

## Phase 3: User Story 1 - Per-Host Activity in Job Log (Priority: P1) 🎯 MVP

**Goal**: Job log report includes a "Recent Host Activity" section showing per-host last event status (ok/failed/unreachable) and timestamp; gracefully shows "No recent job event data available" when no events exist

**Independent Test**: Launch the job template against AAP with completed job history — verify the job log shows the "Recent Host Activity" section with at least one host entry showing a status and timestamp. Then run against an instance with no prior job history — verify the job succeeds with the graceful message.

### Implementation for User Story 1

- [x] T004 [US1] Add "Fetch job events page 1" task to `playbooks/count_managed_nodes.yml` after T003: use `ansible.builtin.uri` with `url: "{{ api_base }}/job_events/?job__in={{ recent_job_ids_str }}&host_name__gt=&event__startswith=runner_on&order_by=-created&page_size=200"`, same auth/validate pattern, `status_code: [200]`, `failed_when: false`, `changed_when: false`; register as `job_events_page1`; guard with `when: recent_job_ids | length > 0`

- [x] T005 [US1] Add "Set initial job events accumulator" task to `playbooks/count_managed_nodes.yml` after T004: use `ansible.builtin.set_fact` to set `all_job_events: "{{ job_events_page1.json.results | default([]) }}"` and `job_events_total_pages: "{{ [(job_events_page1.json.count | default(0) / 200) | round(0, 'ceil') | int, 1] | max }}"` ; guard with `when: recent_job_ids | length > 0`

- [x] T006 [US1] Add "Fetch remaining job event pages" task to `playbooks/count_managed_nodes.yml` after T005: use `ansible.builtin.uri` with `url: "{{ api_base }}/job_events/?job__in={{ recent_job_ids_str }}&host_name__gt=&event__startswith=runner_on&order_by=-created&page_size=200&page={{ item }}"`, same auth pattern; loop `range(2, job_events_total_pages | int + 1) | list`; register as `job_events_remaining`; guard with `when: recent_job_ids | length > 0 and job_events_total_pages | int > 1`

- [x] T007 [US1] Add "Accumulate all job event pages" task to `playbooks/count_managed_nodes.yml` after T006: use `ansible.builtin.set_fact` to append `job_events_remaining.results | map(attribute='json.results') | flatten | list` to `all_job_events`; guard with `when: recent_job_ids | length > 0 and job_events_total_pages | int > 1`

- [x] T008 [US1] Add "Get unique event hostnames" task to `playbooks/count_managed_nodes.yml` after T007: use `ansible.builtin.set_fact` to set `event_hosts: "{{ all_job_events | default([]) | map(attribute='host_name') | map('lower') | select('ne', '') | unique | list }}"` ; guard with `when: recent_job_ids | length > 0`

- [x] T009 [US1] Add "Build per-host last-status list" task to `playbooks/count_managed_nodes.yml` after T008: use `ansible.builtin.set_fact` in a loop over `event_hosts` to build `host_activity_list` as a list of dicts with `hostname`, `last_status` (mapped from event type: `runner_on_ok`/`runner_on_changed` → `ok`, `runner_on_failed` → `failed`, `runner_on_unreachable` → `unreachable`, `runner_on_skipped` → `skipped`, other → `unknown`), and `last_seen` (the `created` field of the most recent event for that host — use `| selectattr('host_name', 'equalto', item) | sort(attribute='created', reverse=true) | first`); guard with `when: recent_job_ids | length > 0 and event_hosts | length > 0`

- [x] T010 [US1] Add "Build job event aggregates" task to `playbooks/count_managed_nodes.yml` after T009: use `ansible.builtin.set_fact` to set `job_events_available: true`, `hosts_with_recent_failures: "{{ host_activity_list | selectattr('last_status', 'in', ['failed', 'unreachable']) | list | length }}"`, `hosts_with_recent_success: "{{ host_activity_list | selectattr('last_status', 'equalto', 'ok') | list | length }}"`; guard with `when: recent_job_ids | length > 0 and event_hosts | length > 0`

- [x] T011 [US1] Update the "Build count result" `ansible.builtin.set_fact` task in `playbooks/count_managed_nodes.yml` to add three new fields: `job_events_available: "{{ job_events_available }}"`, `hosts_with_recent_failures: "{{ hosts_with_recent_failures }}"`, `hosts_with_recent_success: "{{ hosts_with_recent_success }}"`

- [x] T012 [US1] Update the "Print managed node count report" `ansible.builtin.debug` task in `playbooks/count_managed_nodes.yml` to append the "Recent Host Activity" section after the Host Metrics block, matching the contract in `specs/003-job-events-hosts/contracts/playbook-interface.md`: when `count_result.job_events_available` is true, list each entry from `host_activity_list` showing `hostname`, `last_status` (padded for alignment), and `last_seen`; show failure/success totals; when false, show "No recent job event data available"

**Checkpoint**: User Story 1 complete — launch job and verify "Recent Host Activity" section in job log

---

## Phase 4: User Story 2 - Per-Host Aggregates in CSV (Priority: P2)

**Goal**: CSV files in GitHub include `hosts_with_recent_failures` and `hosts_with_recent_success` columns; empty when event data unavailable

**Independent Test**: After a successful run with job history, pull the CSV from the GitHub `output/` directory and confirm it has 13 columns with the header ending in `...,hosts_with_recent_failures,hosts_with_recent_success` and non-empty integer values in those columns.

### Implementation for User Story 2

- [x] T013 [US2] Update the "Build CSV content for GitHub" `ansible.builtin.set_fact` task in `playbooks/count_managed_nodes.yml` to extend `csv_content_raw` with two new columns appended after `most_recent_automation`: `{{ count_result.hosts_with_recent_failures }},{{ count_result.hosts_with_recent_success }}`

- [x] T014 [US2] Update the "Write CSV report" `ansible.builtin.copy` task in `playbooks/count_managed_nodes.yml` to extend both the header row and data row with the two new columns in the same order as T013

- [x] T015 [US2] Update the "Publish count result as AAP artifacts" `ansible.builtin.set_stats` task in `playbooks/count_managed_nodes.yml` to add three new fields: `managed_node_count_job_events_available`, `managed_node_count_hosts_with_failures`, `managed_node_count_hosts_with_success`

**Checkpoint**: User Stories 1 and 2 complete — job log shows activity section AND CSV has 13 columns in GitHub

---

## Phase 5: Polish & Cross-Cutting Concerns

**Purpose**: Documentation updates, linting, and end-to-end validation

- [x] T016 [P] Update `docs/README.md` to add a "Recent Host Activity" section explaining the per-host event enrichment, noting it requires at least one prior completed job run and gracefully degrades when no history exists

- [x] T017 [P] Update `docs/limitations.md` to add a caveat that job event history may be purged by AAP's data retention policy (`AwxTask` cleanup jobs), causing the section to show "No recent job event data available" even on active systems; also note that only `runner_on_*` events are included (gather_facts, module calls) — playbook-level events are excluded

- [x] T018 Run `ansible-lint playbooks/count_managed_nodes.yml` from repository root and fix all reported issues (profile: production)

- [ ] T019 Run quickstart.md validation scenarios end-to-end against a live AAP 2.x instance: verify "Recent Host Activity" section in job log (Scenario 1), graceful degradation with no history (Scenario 2), and 13-column CSV in GitHub (verification checklist)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — read-only review
- **Foundational (Phase 2)**: Depends on Phase 1; T002 → T003 sequential
- **User Story 1 (Phase 3)**: Depends on Phase 2; T004 → T005 → T006 → T007 → T008 → T009 → T010 → T011 → T012 sequential
- **User Story 2 (Phase 4)**: Depends on Phase 3 (needs updated `count_result` from T011)
- **Polish (Phase 5)**: T016 and T017 after Phase 3; T018 after all playbook tasks; T019 after T018

### User Story Dependencies

- **US1 (P1)**: Depends only on Foundational (Phase 2)
- **US2 (P2)**: Depends on US1 (needs `count_result` updated in T011)

---

## Parallel Execution Examples

### Phase 5 Parallels (documentation)
```
Task: "Update docs/README.md with Recent Host Activity section"  → T016
Task: "Update docs/limitations.md with event retention caveat"   → T017 (parallel with T016)
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup (review)
2. Complete Phase 2: Foundational (job IDs fetch + defaults)
3. Complete Phase 3: User Story 1 (per-host activity in job log)
4. **STOP and VALIDATE**: Launch job — confirm "Recent Host Activity" section appears
5. Deploy/demo if ready

### Incremental Delivery

1. Setup + Foundational → job IDs fetched, defaults set
2. User Story 1 → per-host activity visible in job log (MVP)
3. User Story 2 → aggregate counts in CSV and GitHub output
4. Polish → docs updated, lint clean, end-to-end validated

---

## Notes

- All changes are in-place edits to `playbooks/count_managed_nodes.yml` — no new files
- T004–T010 are all guarded with `when: recent_job_ids | length > 0` — no execution when no prior jobs exist
- The per-host loop (T009) uses `selectattr` + `sort` + `first` — works correctly in ansible-core 2.15+
- `host_name__gt=` (greater than empty string) filter excludes playbook-level events server-side
- `order_by=-created` on the events query ensures most recent events are on page 1, reducing need for full pagination on most installations
