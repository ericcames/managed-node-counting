# Quickstart: Job Events Per-Host Enrichment

**Branch**: `003-job-events-hosts` | **Date**: 2026-04-17

---

## Prerequisites

- Features 001 and 002 deployed and working on your AAP instance.
- At least one completed job run (other than the count playbook itself) with
  host-level events in the AAP job history.
- "Count Managed Nodes" job template configured as per existing setup.

---

## Scenario 1: Job Event Data Available

**Setup**: AAP instance with completed job runs in history.

**Steps**:
1. Sync the project to pick up the updated playbook.
2. Launch the "Count Managed Nodes" job template.
3. In the job log, verify the "Recent Host Activity" section appears after
   the Host Metrics block:
   ```
   Recent Host Activity (last 10 jobs)
   <hostname>   : ok/failed/unreachable  (<timestamp>)
   ...
   Hosts with failures : <N>
   Hosts successful    : <N>
   ```
4. Pull the latest CSV from GitHub `output/` and verify it has 13 columns
   with non-empty values in the last two.

**Expected result**: Job succeeds; activity section populated; CSV has 13 columns.

---

## Scenario 2: No Job Event History

**Setup**: New AAP installation or events purged.

**Steps**:
1. Launch the job template.
2. Verify the job log shows:
   ```
   Recent Host Activity : No recent job event data available
   ```
3. Verify job exits **Successful**.
4. Pull CSV — verify last two columns are empty but present.

**Expected result**: Job succeeds with graceful message; CSV schema consistent.

---

## Scenario 3: Only the Count Job Has Run

**Setup**: The only completed jobs are previous runs of the count playbook itself.
The count job is excluded from its own event query.

**Steps**:
1. Launch the job.
2. Verify the job log shows the "No recent job event data available" message
   (because the count job's own events are excluded).

**Expected result**: Job succeeds gracefully.

---

## Verification Checklist

- [ ] "Recent Host Activity" section appears in job log on AAP with job history
- [ ] Per-host status correctly shows ok / failed / unreachable
- [ ] Job succeeds gracefully when no event data available
- [ ] CSV in GitHub has 13 columns with correct header
- [ ] `hosts_with_recent_failures` + `hosts_with_recent_success` counts are correct
- [ ] Job run time increase is ≤10 seconds compared to pre-feature baseline
