# Research: Ansible Managed Node Count Playbook

**Branch**: `001-ansible-node-count` | **Date**: 2026-04-17
**Input**: Feature spec + AAP support lifecycle policy + Red Hat article 3331481

---

## Decision 1: API Approach for Counting Managed Nodes

**Decision**: Use `ansible.builtin.uri` to call the AAP/Tower REST API endpoint
`GET /api/v2/hosts/?page_size=1` and read the `count` field from the response.

**Rationale**: `ansible.builtin.uri` is part of `ansible.builtin`, which is
Red Hat certified content included with every AAP installation. No additional
collection install is required. The `/api/v2/hosts/` endpoint returns a paginated
response whose top-level `count` field reflects the total number of unique hosts
in the controller database — matching the licensing definition of managed nodes.

**Alternatives considered**:
- `ansible.controller` collection: Red Hat certified but requires collection
  install; overkill for a single API call. Considered and rejected in favour of
  `ansible.builtin.uri` for simplicity (Constitution Principle V).
- `awx.awx` collection: Community (Ansible Galaxy) — rejected because it is not
  Red Hat certified or validated content (violates FR-007).
- Direct SSH to controller database: Rejected — unsupported, fragile, and
  bypasses API contract.

---

## Decision 2: Credential Injection Without Extra Vars

**Decision**: Define a Custom Credential Type in AAP/Tower that injects the
following environment variables when attached to the job template:
- `CONTROLLER_HOST` — the FQDN/IP of the AAP/Tower controller
- `CONTROLLER_OAUTH_TOKEN` — an OAuth2 personal access token with read access
  to the Hosts API

**Rationale**: AAP and Tower both support Custom Credential Types that can inject
environment variables into the job execution environment without requiring extra
vars in the job template. This satisfies FR-002 (no extra vars) and FR-001
(credential assertion via `ansible.builtin.assert` checking env var presence).
Environment variable names follow the `ansible.controller` collection convention,
making future migration to that collection straightforward.

**Alternatives considered**:
- Machine credential: Designed for SSH access to managed nodes, not API
  authentication — rejected as wrong credential type for this use case.
- Extra vars: Explicitly prohibited by user requirement (FR-002).
- Vault-encrypted vars in the project: Requires extra setup and doesn't integrate
  cleanly with AAP's credential management UI — rejected.

---

## Decision 3: Supported AAP/Tower Versions

**Decision**: Primary support targets AAP 2.4, 2.5, and 2.6. Ansible Tower
is documented as legacy with a best-effort compatibility note.

**Rationale**: Per Red Hat's Ansible Automation Platform support lifecycle
(access.redhat.com/support/policy/updates/ansible-automation-platform):
- AAP 2.6: Full Support, ansible-core 2.16
- AAP 2.5: Full Support, ansible-core 2.16
- AAP 2.4: Maintenance Support, ansible-core 2.15, Extended Update Support
  add-on available
- Ansible Tower: Legacy, no longer listed in current AAP lifecycle documentation

The `/api/v2/` REST API is consistent across AAP 2.4–2.6 and Ansible Tower 3.x.
Playbook will target ansible-core 2.15 minimum (AAP 2.4 baseline) to maximise
compatibility.

**Alternatives considered**:
- Target only AAP 2.5+: Excludes customers on Maintenance Support (AAP 2.4) —
  rejected.
- Drop Tower support entirely: User requirement includes Tower in the README —
  retained as best-effort with clear limitations documented.

---

## Decision 4: CSV Output Location and Strategy

**Decision**: Write the CSV file to `output/managed_node_count.csv` within the
project's working directory on the execution node. The CSV is overwritten on each
run (not appended). Manual git commit is required to persist it to the SCM repo.

**Rationale**: AAP/Tower job execution checks out the SCM project to a working
directory on the execution node. The playbook can write files to this directory
using `ansible.builtin.copy` or `ansible.builtin.template`. This is the simplest
approach satisfying FR-005 without requiring git credentials inside the playbook.

**Limitation**: The file is written to the execution node, not automatically
committed to the git repository. Operators must retrieve the file (via AAP
artifacts, fetch to controller, or manual checkout) and commit it separately.
This limitation is documented in the README (FR-009).

**Alternatives considered**:
- Auto-commit via `ansible.builtin.git`: Requires a separate git credential
  injected into the playbook — adds complexity, violates Principle V.
- AAP artifact storage (survey output): Not designed for structured file output.
- Append to CSV instead of overwrite: Makes idempotency harder to verify
  (Constitution Principle II) — rejected; overwrite is simpler and correct.

---

## Decision 5: Node Count Definition

**Decision**: Count is sourced from `GET /api/v2/hosts/?page_size=1` → `.count`
field, which reflects unique hosts in the controller database. This aligns with
Red Hat's licensing definition: each distinct identifiable instance = 1 node,
regardless of multiple inventory memberships.

**Rationale**: Per Red Hat article 3331481, the managed node count rules are:
- Same host accessed via multiple paths = 1 node (deduplicated by controller)
- No cycling/rotation to exceed subscription limits
- Count cannot be manipulated by clearing tracking data
The AAP/Tower API's host count already applies this deduplication — it is the
authoritative source for licensing counts.

**Alternatives considered**:
- Count inventory hosts: Would double-count hosts appearing in multiple
  inventories — rejected as incorrect per licensing definition.
- Count active job connections: Point-in-time, not accurate for licensing
  purposes — rejected.
