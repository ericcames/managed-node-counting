# Feature Specification: Ansible Managed Node Count Playbook

**Feature Branch**: `001-ansible-node-count`
**Created**: 2026-04-17
**Status**: Draft
**Input**: Count managed nodes in Ansible Automation Platform or Ansible Tower via GUI execution

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Count Managed Nodes via AAP/Tower GUI (Priority: P1)

An operator opens the Ansible Automation Platform (or Ansible Tower) web interface,
launches the "Count Managed Nodes" job template, and within a single job run sees the
total count of managed nodes printed clearly in the job output log.

**Why this priority**: This is the primary deliverable — without an accurate count
in the job log, the playbook has no value.

**Independent Test**: Launch the job template in AAP or Tower GUI with no extra
variables configured. The job MUST complete successfully and print a node count.

**Acceptance Scenarios**:

1. **Given** the job template has an appropriate credential attached and no extra vars
   configured, **When** the operator launches the job from the GUI, **Then** the first
   task asserts the credential is present and the job log shows the total managed
   node count before the job completes.

2. **Given** the job template has NO appropriate credential attached, **When** the
   operator launches the job, **Then** the first assert task FAILS with a clear error
   message identifying which credential is missing.

3. **Given** the playbook is run multiple times in succession without changing the
   environment, **When** each run completes, **Then** each run reports the same node
   count (idempotent).

---

### User Story 2 - CSV Report Written to Software Repository (Priority: P2)

After a successful job run, a CSV file is written to the software repository location
so that operators and compliance teams can retrieve the count data for reporting,
auditing, or license compliance purposes.

**Why this priority**: The job log is ephemeral; the CSV provides a persistent,
machine-readable record for downstream use.

**Independent Test**: After a successful job run, a CSV file exists in the expected
repository location containing the run timestamp, controller URL, and node count.

**Acceptance Scenarios**:

1. **Given** a successful job run completes, **When** the operator inspects the
   expected output path in the repository, **Then** a CSV file exists with columns
   for timestamp, controller URL, total node count, and enabled/disabled breakdown.

2. **Given** the playbook is run a second time, **When** it completes, **Then** the
   CSV file is updated (overwritten) with the latest run data and no duplicate rows
   are introduced from a single run.

---

### User Story 3 - Operator Sets Up Playbook from Documentation (Priority: P3)

A new operator follows the README to configure the job template in either AAP (2.4,
2.5, or 2.6) or Ansible Tower and successfully runs the playbook for the first time.

**Why this priority**: The playbook's value depends on operators being able to
configure it without external assistance.

**Independent Test**: A new operator with no prior context can read the README and
successfully execute the playbook within 30 minutes.

**Acceptance Scenarios**:

1. **Given** an operator with AAP access follows the README setup steps, **When** they
   launch the job template, **Then** the first run completes successfully without
   needing to consult additional documentation.

2. **Given** an operator using Ansible Tower (legacy) follows the Tower-specific
   README section, **When** they launch the job, **Then** the playbook runs
   successfully or displays a clear compatibility warning.

---

### Edge Cases

- What happens when the controller API is unreachable at job execution time?
- How does the playbook handle zero managed nodes in the inventory?
- What happens when the credential has insufficient API permissions to query hosts?
- What if the output directory for the CSV does not exist or is not writable?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The playbook MUST execute an assert task as its FIRST task to verify
  that the required credential environment variables are present before any other
  logic runs.
- **FR-002**: The playbook MUST NOT require any extra variables to be defined in the
  job template configuration.
- **FR-003**: The playbook MUST count all managed nodes as defined by Red Hat Ansible
  Automation Platform licensing terms (each uniquely identifiable host = 1 node,
  regardless of how many inventories it appears in).
- **FR-004**: Count results MUST be written to the job execution log in a
  human-readable format, including total count and enabled/disabled breakdown.
- **FR-005**: Count results MUST be written to a CSV file at a defined path within
  the project's software repository directory.
- **FR-006**: The playbook MUST be compatible with Ansible Automation Platform
  versions 2.4, 2.5, and 2.6.
- **FR-007**: All tasks MUST use only Red Hat certified or validated Ansible content
  (no community or unsupported collections).
- **FR-008**: The repository MUST include a README with step-by-step setup
  instructions for both AAP and Ansible Tower.
- **FR-009**: The README MUST include a dedicated section documenting known
  limitations of the playbook.
- **FR-010**: The playbook MUST be runnable entirely from the AAP/Tower web
  interface without any command-line access.

### Key Entities *(include if feature involves data)*

- **ManagedNode**: A uniquely identifiable host being managed by AAP/Tower. Defined
  by Red Hat licensing as any Virtual Node, Physical Node, container, appliance,
  software instance, or networking controller directly or indirectly managed.
- **NodeCountResult**: The output of a single playbook run — timestamp, controller
  URL, total count, enabled count, disabled count.
- **CSVReport**: A persisted, comma-separated record of one or more NodeCountResult
  entries written to the project repository.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: 100% of job runs with a valid credential attached complete successfully
  and report an accurate node count in the job log.
- **SC-002**: 100% of job runs with a missing or invalid credential FAIL on the first
  assert task with a descriptive error message before any count logic executes.
- **SC-003**: A CSV file reflecting the most recent run exists in the repository
  location after every successful execution.
- **SC-004**: A new operator can complete the full setup and first successful run
  using only the README within 30 minutes.
- **SC-005**: The playbook runs without modification on AAP 2.4, 2.5, and 2.6.

## Assumptions

- The playbook runs ON the AAP/Tower controller itself (localhost execution), not
  against remote managed nodes.
- The controller API endpoint and authentication token are injected via a custom
  credential type attached to the job template — no extra vars are used.
- Ansible Tower (legacy) support is best-effort; Tower is approaching end-of-life
  and the README will note this.
- CSV output is written to the execution node's project working directory; it is
  NOT automatically committed to git — operators must manually retrieve or commit it.
- The playbook counts hosts via the controller REST API, not by connecting to
  individual managed nodes.
- "Red Hat certified or validated content" means content sourced from
  console.redhat.com/ansible/automation-hub (not Ansible Galaxy).
