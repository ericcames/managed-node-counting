# Feature Specification: Host Metrics Integration

**Feature Branch**: `002-host-metrics`
**Created**: 2026-04-17
**Status**: Draft
**Input**: Extend the managed node count playbook to also pull in host metrics data from the AAP/Tower host_metrics endpoint if it is available. The metrics should be included in the job log report and the CSV output. If the endpoint is not available (older Tower versions), the playbook should gracefully degrade and continue without metrics data.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - View Host Metrics in Job Log Report (Priority: P1)

An operator runs the managed node count job template and sees host metrics data
(automation usage, last automation date, hosts automated vs. not automated) included
in the job log report alongside the existing node count summary. This gives the operator
a richer picture of how managed nodes are being used without needing to navigate
additional screens.

**Why this priority**: Metrics in the job log deliver immediate value with zero extra
steps — the operator already reviews the log output. It is the simplest way to surface
the data and forms the foundation for the CSV enhancement.

**Independent Test**: Launch the job template against an AAP 2.x instance that has
`/api/controller/v2/host_metrics/` available. Confirm the job log prints a metrics
block showing total hosts automated, hosts not automated, and the date range of
automation activity.

**Acceptance Scenarios**:

1. **Given** the job runs against an AAP instance where the host metrics endpoint is
   available, **When** the job completes, **Then** the job log includes a "Host Metrics"
   section showing: total hosts tracked, hosts automated at least once, hosts never
   automated, and the oldest/most recent automation dates.

2. **Given** the job runs against an older Ansible Tower instance where the host metrics
   endpoint does not exist (returns 404), **When** the job completes, **Then** the job
   log prints a note that host metrics are not available on this platform and the job
   succeeds without error.

3. **Given** the job runs against an AAP instance but the credential lacks permission
   to read host metrics, **When** the endpoint returns 403, **Then** the job logs a
   warning that metrics could not be retrieved (permission denied) and continues
   successfully.

---

### User Story 2 - Host Metrics Included in CSV Output (Priority: P2)

After the job runs, the CSV file pushed to the GitHub repository includes additional
columns for host metrics data. Operators and managers can track automation coverage
over time by reviewing the CSV history in the repository.

**Why this priority**: The CSV is the persistent record; including metrics there
enables trending analysis. Depends on User Story 1 (metrics must be retrieved before
they can be written to CSV).

**Independent Test**: After a successful run against an AAP instance with metrics
available, pull the CSV from the GitHub `output/` directory and verify it contains
the additional metrics columns with non-empty values.

**Acceptance Scenarios**:

1. **Given** host metrics are available and retrieved, **When** the CSV is written,
   **Then** the CSV includes additional columns: `hosts_automated`, `hosts_not_automated`,
   `hosts_automated_deleted`, `earliest_automation`, `most_recent_automation`.

2. **Given** host metrics are not available (older Tower), **When** the CSV is written,
   **Then** the additional metrics columns are present but empty, preserving a
   consistent schema across all runs.

---

### Edge Cases

- What happens when the host metrics endpoint returns a non-200/404/403 status (e.g., 500)?
  The job should log the unexpected status code as a warning and continue without metrics.
- What happens when the metrics endpoint returns results but the counts are all zero
  (new installation with no automation history)? Zero values are valid and must be
  written as `0`, not left blank.
- What if pagination is required on the host_metrics endpoint? The spec assumes the
  endpoint returns aggregate counts (not per-host records) so pagination is not expected,
  but the implementation should handle it if encountered.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The playbook MUST attempt to retrieve host metrics from the controller's
  host metrics endpoint after the node count is calculated.
- **FR-002**: If the host metrics endpoint returns a success response, the playbook MUST
  extract: total hosts tracked, count of hosts automated at least once, count of hosts
  never automated, count of deleted hosts that were automated, earliest automation date,
  and most recent automation date.
- **FR-003**: The job log report MUST include a "Host Metrics" section when metrics data
  is successfully retrieved, displaying the values from FR-002 in human-readable form.
- **FR-004**: If the host metrics endpoint is unavailable (404) or returns a permission
  error (403), the playbook MUST log an informational message and continue without
  failing the job.
- **FR-005**: If the host metrics endpoint returns any other unexpected error (e.g., 500),
  the playbook MUST log a warning with the status code and continue without failing.
- **FR-006**: The CSV output MUST include columns for all host metrics fields defined in
  FR-002, appended after the existing columns.
- **FR-007**: When metrics are unavailable, the CSV metrics columns MUST be written as
  empty strings to preserve a consistent schema.
- **FR-008**: No extra variables may be added to the job template. All configuration
  must come from the existing credential types.
- **FR-009**: The playbook MUST work without modification against AAP instances that
  do not have the host metrics endpoint (graceful degradation).

### Key Entities

- **Host Metrics Summary**: Aggregate counts returned by the host metrics endpoint —
  total hosts tracked, automated, not automated, deleted+automated, date range of activity.
- **Enhanced CSV Row**: The existing CSV row extended with host metrics fields; empty
  when metrics are unavailable.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: When run against an AAP instance with host metrics available, 100% of
  job runs include a populated metrics section in the job log with no additional
  operator action.
- **SC-002**: When run against an Ansible Tower instance without the host metrics
  endpoint, 100% of job runs complete successfully with a graceful "not available"
  message — zero job failures attributable to the missing endpoint.
- **SC-003**: The CSV files in the GitHub repository contain a consistent column schema
  across all runs regardless of whether metrics were available.
- **SC-004**: The additional metrics retrieval adds no more than 5 seconds to the total
  job run time.

## Assumptions

- The AAP host metrics endpoint (`/api/controller/v2/host_metrics/`) returns aggregate
  summary data accessible with the same read credential already used for host counts —
  no additional credential fields are required.
- The existing "AAP Node Count Credential" credential type does not need to be modified.
- Ansible Tower instances do not have this endpoint; a 404 response is the expected
  graceful-degradation signal.
- Metrics data is retrieved as a single API call (or small number of pages) — no
  long-running pagination loop is expected for the aggregate summary.
- The GitHub credential type and CSV push mechanism remain unchanged; only additional
  column values are added.
