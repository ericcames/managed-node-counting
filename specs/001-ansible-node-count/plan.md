# Implementation Plan: Ansible Managed Node Count Playbook

**Branch**: `001-ansible-node-count` | **Date**: 2026-04-17 | **Spec**: [spec.md](spec.md)
**Input**: Feature specification from `specs/001-ansible-node-count/spec.md`

## Summary

Build an Ansible playbook (`count_managed_nodes.yml`) that counts the total
managed nodes registered in an Ansible Automation Platform (2.4–2.6) or
Ansible Tower controller via the REST API. The playbook runs from the GUI with
no extra vars — all connection details are injected via a Custom Credential Type.
The first task asserts credential presence; output goes to the job log and a CSV
file in the project repository. All content uses `ansible.builtin` only (certified). The playbook should deduplicate hosts.  Hosts can be entered as IP addresses, fqdn, or hostname.

## Technical Context

**Language/Version**: YAML (Ansible playbook), ansible-core 2.15 minimum (AAP 2.4 baseline)
**Primary Dependencies**: `ansible.builtin` (certified, no additional installs)
**Storage**: `output/managed_node_count.csv` on execution node (manual git commit required)
**Testing**: ansible-lint for syntax/style; manual integration test against developer-supplied AAP/Tower instance (URL, username, password collected in T000 and stored in `docs/dev-environment.md`)
**Target Platform**: Ansible Automation Platform 2.4, 2.5, 2.6; Ansible Tower 3.8.x (legacy)
**Project Type**: Ansible automation content (playbook + documentation)
**Performance Goals**: Playbook completes in under 60 seconds; API pagination handled for any estate size
**Constraints**: No extra vars; certified content only; GUI-executable; output to job log + CSV; requires a dedicated test AAP/Tower instance (developer-provided)
**Scale/Scope**: Single playbook; supports any size Ansible estate (pagination-safe API query)

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Accuracy First | ✅ PASS | Count sourced from authoritative controller API at execution time |
| II. Idempotent Operations | ✅ PASS | Count-only; CSV overwrite (not append); same estate = same count |
| III. Test-First | ⚠️ ADAPTED | Ansible playbooks have limited automated test tooling; ansible-lint + integration test against live controller is the practical equivalent; documented in quickstart.md |
| IV. Observability | ✅ PASS | Structured job log output + CSV with timestamp, controller URL, counts, job ID |
| V. Simplicity | ✅ PASS | Single playbook, `ansible.builtin` only, no collection installs, linear execution |

**Complexity Tracking**: Test-First adaptation is noted above. The practical
equivalent (ansible-lint + integration test) satisfies the intent of Principle III
for an automation content artifact. No traditional unit test framework applies to
YAML playbooks; this is documented as a known limitation.

## Project Structure

### Documentation (this feature)

```text
specs/001-ansible-node-count/
├── plan.md              # This file
├── spec.md              # Feature specification
├── research.md          # Phase 0 research decisions
├── data-model.md        # Entity and CSV schema definitions
├── quickstart.md        # Validation steps
├── contracts/
│   └── playbook-interface.md   # Input/output/compatibility contract
└── checklists/
    └── requirements.md         # Spec quality checklist
```

### Source Code (repository root)

```text
playbooks/
└── count_managed_nodes.yml    # Main playbook

docs/
├── README.md                  # Setup guide: AAP + Tower, Custom Credential Type,
│                              #   Job Template config, known limitations
└── limitations.md             # Standalone limitations reference

output/
└── .gitkeep                   # Placeholder; managed_node_count.csv written here at runtime
```

**Structure Decision**: Single-project layout. No `src/` or `tests/` directories
needed — this is an Ansible content project. Documentation lives in `docs/`;
runtime output in `output/`; playbook in `playbooks/`.

## Complexity Tracking

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|--------------------------------------|
| Test-First adapted | Ansible YAML has no unit test framework | ansible-lint + integration testing is the industry standard for Ansible content; writing a Python test harness would violate Principle V (unnecessary complexity) |
