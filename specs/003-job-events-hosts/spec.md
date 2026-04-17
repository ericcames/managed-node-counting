# Feature Specification: Job Events Per-Host Enrichment

**Feature Branch**: `003-job-events-hosts`
**Created**: 2026-04-17
**Status**: Draft
**Input**: Extend the managed node count playbook to query the AAP job_events endpoint to pull per-host event data from recent job runs, enriching the report with which hosts were recently targeted, what tasks ran against them, and per-host success/failure status.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - View Per-Host Automation Activity in Job Log (Priority: P1)

An operator runs the managed node count job template and sees, in the job log,
a summary of recent per-host automation activity pulled from job events. The
summary shows each unique host that has appeared in recent job runs, along with
whether its last event was a success or failure. This gives the operator a
quick view of automation health across the fleet without navigating to individual
job histories.

**Why this priority**: The job log summary is the fastest way to surface
per-host activity data. It requires no additional tooling and delivers
immediate operational value: operators can spot hosts with persistent failures
at a glance.

**Independent Test**: Launch the job template against an AAP instance with
recent job history. Confirm the job log prints a "Recent Host Activity" section
listing hosts, their most recent job event status (success/failure/unreachable),
and the date of their last automation event.

**Acceptance Scenarios**:

1. **Given** the job runs against an AAP instance with job event history,
   **When** the job completes, **Then** the job log includes a "Recent Host
   Activity" section showing for each host: hostname, last event status
   (ok/failed/unreachable), and timestamp of last automation event.

2. **Given** the job runs and some hosts have only successful events,
   **When** the job completes, **Then** those hosts show status "ok" in the
   report.

3. **Given** the job runs and some hosts have a most recent event that was
   a failure or unreachable, **When** the job completes, **Then** those hosts
   are clearly flagged (e.g., "failed" or "unreachable") in the report.

4. **Given** the controller has no job event history (new installation or
   events have been purged), **When** the job completes, **Then** the report
   prints "No recent host activity found" and the job succeeds without error.

5. **Given** the job runs against Ansible Tower or an older AAP version where
   the job_events endpoint behaves differently, **When** the endpoint returns
   an error or empty results, **Then** the job succeeds with a graceful
   degradation message.

---

### User Story 2 - Per-Host Activity Summary in CSV Output (Priority: P2)

After each run, the CSV pushed to the GitHub repository includes a count of
hosts with recent successful events and a count with recent failures, giving
managers a high-level automation health metric over time.

**Why this priority**: Trends in per-host failure rates are a key operational
metric. The CSV provides the persistent record for tracking this over time.
Per-host detail rows are not added to the CSV (that would create a
one-to-many structure); instead, aggregate counts are appended as new columns.

**Independent Test**: After a successful run, pull the CSV from the GitHub
`output/` directory and verify it contains two new columns:
`hosts_with_recent_failures` and `hosts_with_recent_success`, with integer
values reflecting the counts from the job log report.

**Acceptance Scenarios**:

1. **Given** job event data is available, **When** the CSV is written, **Then**
   it includes `hosts_with_recent_failures` and `hosts_with_recent_success`
   columns with correct integer counts.

2. **Given** no job event data is available, **When** the CSV is written,
   **Then** the two new columns are present but empty, preserving schema
   consistency.

---

### Edge Cases

- What if the job_events endpoint returns thousands of events (large fleet,
  long history)? The query must be bounded to a recent time window or limited
  page count to avoid excessive runtime.
- What if a host appears in events with mixed success/failure statuses? The
  report uses the most recent event status for that host.
- What if event hostnames differ in case from the hosts/ endpoint (e.g.,
  `Web01` vs `web01`)? Normalize to lowercase for consistency.
- What if the job currently running is the only job and its events haven't
  finished writing? Exclude the current job's events or accept partial data
  with a note.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The playbook MUST query the controller's job events endpoint to
  retrieve per-host event data from recent job runs.
- **FR-002**: The query MUST be bounded — fetching only the most recent N job
  runs (default: last 10 jobs) or events within the last 30 days, whichever
  is smaller, to keep runtime bounded.
- **FR-003**: For each unique host found in the event data, the playbook MUST
  determine the most recent event status: success (ok/changed), failure
  (failed), or unreachable.
- **FR-004**: The job log MUST include a "Recent Host Activity" section listing
  each unique host with its last event status and last event timestamp.
- **FR-005**: If the job events endpoint returns no data or is unavailable,
  the playbook MUST log an informational message and continue without failing.
- **FR-006**: The CSV output MUST include two new aggregate columns:
  `hosts_with_recent_failures` (count of hosts whose last event was a failure
  or unreachable) and `hosts_with_recent_success` (count whose last event was
  ok or changed).
- **FR-007**: When job event data is unavailable, the two new CSV columns MUST
  be written as empty strings to preserve schema consistency.
- **FR-008**: No extra variables may be added to the job template. No new
  credential types are required — the existing controller credential provides
  sufficient access.
- **FR-009**: Host names in event data MUST be normalized to lowercase before
  deduplication and comparison.

### Key Entities

- **HostActivitySummary**: Per-host rollup — hostname (normalized), last event
  status, last event timestamp. Derived from job event records.
- **JobEventAggregates**: Two scalar counts appended to the existing CSV row —
  hosts with recent failures, hosts with recent success.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: When run against an AAP instance with job history, 100% of job
  runs include a populated "Recent Host Activity" section in the job log with
  no additional operator action.
- **SC-002**: When run against an instance with no event history or an
  unavailable endpoint, 100% of job runs complete successfully with a graceful
  message — zero failures attributable to missing event data.
- **SC-003**: The CSV files in the GitHub repository contain a consistent
  13-column schema (11 existing + 2 new) across all runs regardless of
  whether event data was available.
- **SC-004**: The additional job events query adds no more than 10 seconds to
  the total job run time across fleets up to 500 hosts.
- **SC-005**: The "Recent Host Activity" section correctly identifies hosts
  with failures — verified by comparing report output against known job
  history on the test instance.

## Assumptions

- The existing controller credential (OAuth2 token or username/password) has
  read access to the job_events endpoint — no new credential fields are needed.
- "Recent" is defined as the last 10 completed jobs on the controller; this
  provides a meaningful sample without unbounded pagination.
- The job_events endpoint returns individual task-level events per host; the
  playbook aggregates these to a per-host last-status summary.
- The current running job (job ID from `job_id` fact) is excluded from the
  event query to avoid partial/in-progress data contaminating the report.
- Per-host detail rows are NOT added to the CSV — only aggregate counts are
  added to keep the one-row-per-run CSV schema intact.
- The feature depends on features 001 (node count) and 002 (host metrics)
  already being present in the playbook.
