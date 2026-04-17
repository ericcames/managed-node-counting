---
description: "Task list for ansible managed node count playbook"
---

# Tasks: Ansible Managed Node Count Playbook

**Input**: Design documents from `specs/001-ansible-node-count/`
**Prerequisites**: plan.md ✅, spec.md ✅, research.md ✅, data-model.md ✅, contracts/playbook-interface.md ✅

**Tests**: Not requested — ansible-lint validation included in Polish phase.

**Organization**: Tasks are grouped by user story to enable independent implementation and testing.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: Which user story this task belongs to (US1, US2, US3)
- Include exact file paths in all descriptions

## Path Conventions

- Playbook: `playbooks/`
- Documentation: `docs/`
- Runtime output: `output/`
- Specs/design: `specs/001-ansible-node-count/`

---

## Phase 0: Developer Environment (Pre-Implementation)

**Purpose**: Collect test instance details from the developer before any code is written

**⚠️ REQUIRED**: Complete before Phase 1 — all validation tasks depend on these values

- [x] T000 Prompt the developer for the following test instance details and record them in `docs/dev-environment.md` (gitignored): (1) **Controller URL** — the full HTTPS URL of the AAP or Ansible Tower instance available for testing (e.g. `https://aap.example.com`); (2) **Username** — an account with Org Admin or System Auditor permissions on that instance; (3) **Password** — the password for that account. Confirm the instance is reachable before proceeding.

**Checkpoint**: `docs/dev-environment.md` exists with all three values confirmed reachable

---

## Phase 1: Setup (Shared Infrastructure)

**Purpose**: Create project directory skeleton and shared configuration

- [x] T001 Create project directory structure: `playbooks/`, `docs/`, `output/` at repository root
- [x] T002 Create `output/.gitkeep` placeholder file so the output directory is tracked by git
- [x] T003 Create `.ansible-lint` configuration file at repository root with rules appropriate for ansible-core 2.15 (profile: production, skip: yaml[line-length])
- [x] T004 [P] Create `docs/credential-type.yml` containing the Custom Credential Type input and injector YAML definitions (from `specs/001-ansible-node-count/contracts/playbook-interface.md`)

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: Core playbook scaffolding that all user stories depend on

**⚠️ CRITICAL**: No user story implementation can begin until this phase is complete

- [x] T005 Create `playbooks/count_managed_nodes.yml` with playbook header: `hosts: localhost`, `gather_facts: false`, `connection: local`, and empty `tasks:` list
- [x] T006 Add assert task as the FIRST task in `playbooks/count_managed_nodes.yml`: use `ansible.builtin.assert` to verify `CONTROLLER_HOST` env var is non-empty; fail_msg must identify the missing variable by name and instruct operator to attach the credential
- [x] T007 Add second assert condition to the same task in `playbooks/count_managed_nodes.yml` to verify at least one of `CONTROLLER_OAUTH_TOKEN` or (`CONTROLLER_USERNAME` + `CONTROLLER_PASSWORD`) env vars are present; fail_msg must distinguish which auth method is missing

**Checkpoint**: Assert block complete — job fails fast on missing credentials before any API calls

---

## Phase 3: User Story 1 - Count Managed Nodes via GUI (Priority: P1) 🎯 MVP

**Goal**: Operator launches job from AAP/Tower GUI; job log shows accurate deduplicated node count

**Independent Test**: Launch job template in AAP/Tower with valid credential attached, no extra vars. Job log must print "Managed Node Count Report" block with total, enabled, and disabled counts.

### Implementation for User Story 1

- [x] T008 [US1] Add paginated API fetch task in `playbooks/count_managed_nodes.yml`: use `ansible.builtin.uri` with `url: "{{ lookup('env','CONTROLLER_HOST') }}/api/v2/hosts/"`, `headers: Authorization: "Bearer {{ lookup('env','CONTROLLER_OAUTH_TOKEN') }}"`, loop with `next` link until `next` is null; accumulate all host records into `all_hosts_raw` fact
- [x] T009 [US1] Add set_fact task in `playbooks/count_managed_nodes.yml` to extract host name list from `all_hosts_raw` and normalize to lowercase: `all_host_names: "{{ all_hosts_raw | map(attribute='name') | map('lower') | list }}"`
- [x] T010 [US1] Add set_fact task in `playbooks/count_managed_nodes.yml` to deduplicate the name list: `deduplicated_hosts: "{{ all_host_names | unique | list }}"` — this handles case-insensitive duplicates (IP vs FQDN deduplication is documented as a limitation)
- [x] T011 [US1] Add second API fetch task in `playbooks/count_managed_nodes.yml` to query `/api/v2/hosts/?enabled=true&page_size=1` and capture `.count` as `enabled_count_raw`; compute `disabled_count` as `deduplicated_hosts | length - enabled_count_raw`
- [x] T012 [US1] Add set_fact task in `playbooks/count_managed_nodes.yml` to build `count_result` dict: `timestamp` (ansible_date_time.iso8601), `controller_url` (CONTROLLER_HOST env var), `total_count` (deduplicated_hosts | length), `enabled_count`, `disabled_count`, `job_id` (AWX_JOB_ID env var or "N/A")
- [x] T013 [US1] Add `ansible.builtin.debug` task in `playbooks/count_managed_nodes.yml` to print the formatted "Managed Node Count Report" block to the job log matching the output format defined in `specs/001-ansible-node-count/contracts/playbook-interface.md`

**Checkpoint**: User Story 1 is fully functional — launch job in GUI and verify job log output

---

## Phase 4: User Story 2 - CSV Report Written to Repository (Priority: P2)

**Goal**: After each successful run, a CSV file exists at `output/managed_node_count.csv` with the run results

**Independent Test**: After successful job run, verify `output/managed_node_count.csv` contains correct header row and one data row matching job log counts.

### Implementation for User Story 2

- [x] T014 [US2] Add `ansible.builtin.copy` task in `playbooks/count_managed_nodes.yml` to write CSV header + data row to `output/managed_node_count.csv` using the schema from `specs/001-ansible-node-count/data-model.md`: columns `timestamp,controller_url,total_count,enabled_count,disabled_count,job_id`; file is overwritten (not appended) each run
- [x] T015 [US2] Add `ansible.builtin.debug` task in `playbooks/count_managed_nodes.yml` to print CSV write confirmation to job log including the resolved file path

**Checkpoint**: User Stories 1 and 2 complete — job log shows count report AND CSV is written to `output/`

---

## Phase 5: User Story 3 - Operator Setup via Documentation (Priority: P3)

**Goal**: New operator follows README to configure and successfully run the playbook on AAP or Ansible Tower

**Independent Test**: A new operator with only README access can complete setup and first successful run within 30 minutes.

### Implementation for User Story 3

- [x] T016 [P] [US3] Write `docs/README.md` — AAP setup section: prerequisites, step-by-step Custom Credential Type creation (include YAML snippets from `docs/credential-type.yml`), Credential creation, Job Template configuration (inventory, project, playbook path, credential attachment, no extra vars), project sync, job launch, and expected job log output
- [x] T017 [P] [US3] Write `docs/README.md` — Ansible Tower setup section (after AAP section): Tower-specific UI differences, `/api/v1/` vs `/api/v2/` note, legacy support caveat, and pointer to limitations.md
- [x] T018 [P] [US3] Write `docs/limitations.md` with the following documented limitations: (1) CSV not auto-committed to git — manual retrieval required; (2) Deduplication is case-insensitive name-only — same host registered as IP and FQDN counts as 2 nodes; (3) Ansible Tower is legacy/end-of-life — best-effort only; (4) Count reflects state at execution time — no historical trending; (5) Credential must have API read access to `/api/v2/hosts/`; (6) Large estates may affect performance (pagination handled but runtime grows linearly); (7) `enabled_count` comes from a separate API call — minor race condition possible in dynamic environments
- [x] T019 [US3] Add "Known Limitations" section to `docs/README.md` summarising `docs/limitations.md` with a link to the full document

**Checkpoint**: All three user stories complete — playbook functional, CSV output working, documentation ready

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: Quality gates, cross-cutting cleanup, and final validation

- [x] T020 [P] Add `.gitignore` at repository root; decide whether to track `output/managed_node_count.csv` — if not tracked, add `output/managed_node_count.csv` to `.gitignore` and update `docs/limitations.md` accordingly; if tracked, document the manual commit workflow in `docs/README.md`
- [x] T021 Run `ansible-lint playbooks/count_managed_nodes.yml` from repository root and fix all reported issues (profile: production)
- [x] T022 [P] Add `docs/README.md` — top-level overview section (project purpose, supported platforms table from contract, quick-start link)
- [ ] T023 Run quickstart.md validation steps end-to-end against a live AAP instance (or document as manual validation required if no instance available)
- [x] T024 [P] Update `specs/001-ansible-node-count/research.md` with any implementation discoveries (e.g. actual API response shape, pagination behaviour observed)

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: No dependencies — start immediately; T003 and T004 can run in parallel
- **Foundational (Phase 2)**: Depends on Phase 1 completion — T005 → T006 → T007 (sequential within phase)
- **User Story 1 (Phase 3)**: Depends on Phase 2; T008–T013 must run in order (each builds on prior fact)
- **User Story 2 (Phase 4)**: Depends on Phase 3 completion (needs `count_result` fact established in T012)
- **User Story 3 (Phase 5)**: Can start after Phase 1 (T016–T018 are documentation-only, fully independent); T019 depends on T016–T018
- **Polish (Phase 6)**: T020 can run anytime after Phase 1; T021 requires Phase 3 complete; T022–T024 require all phases complete

### User Story Dependencies

- **US1 (P1)**: Depends only on Foundational (Phase 2)
- **US2 (P2)**: Depends on US1 (needs count_result fact from T012)
- **US3 (P3)**: Documentation phases (T016–T018) are independent of US1/US2 — can proceed in parallel with Phase 3/4

---

## Parallel Execution Examples

### Phase 1 Parallels
```
Task: "Create playbooks/, docs/, output/ directories"       → T001
Task: "Create output/.gitkeep"                              → T002 (depends T001)
Task: "Create .ansible-lint config"                         → T003 (parallel with T002)
Task: "Create docs/credential-type.yml"                     → T004 (parallel with T002, T003)
```

### Phase 5 Parallels (US3 — all documentation)
```
Task: "Write docs/README.md AAP section"                    → T016
Task: "Write docs/README.md Tower section"                  → T017 (parallel with T016)
Task: "Write docs/limitations.md"                           → T018 (parallel with T016, T017)
```

---

## Implementation Strategy

### MVP First (User Story 1 Only)

1. Complete Phase 1: Setup
2. Complete Phase 2: Foundational (assert tasks — BLOCKS all user stories)
3. Complete Phase 3: User Story 1 (count + job log output)
4. **STOP and VALIDATE**: Launch job in AAP GUI; confirm count appears in job log
5. Deploy/demo if ready

### Incremental Delivery

1. Setup + Foundational → credential assertion working
2. User Story 1 → count in job log (MVP demo-able)
3. User Story 2 → CSV output added
4. User Story 3 → documentation complete (ready for wider operator use)
5. Polish → production-ready

---

## Notes

- [P] tasks = different files or no shared state dependencies
- US3 documentation tasks (T016–T018) can be worked in parallel with US1/US2 implementation
- Deduplication (T009–T010) is case-insensitive name normalization only — cross-format deduplication (IP vs FQDN for same host) is a known limitation documented in T018
- No extra vars permitted at any point — all parameterization via credential injection only
- ansible-lint must pass before T023 validation run
