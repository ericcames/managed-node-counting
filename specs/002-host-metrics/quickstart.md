# Quickstart: Host Metrics Integration

**Branch**: `002-host-metrics` | **Date**: 2026-04-17

---

## Prerequisites

- Feature 001 (`001-ansible-node-count`) fully deployed and working on your AAP instance.
- AAP 2.4 or later (host_metrics endpoint required for US1/US2; Tower graceful degradation tested separately).
- "Count Managed Nodes" job template already configured with the AAP Node Count Credential
  and GitHub Repository Credential attached.

---

## Scenario 1: Metrics Available (AAP 2.4+)

**Setup**: AAP 2.x instance with the host_metrics endpoint available.

**Steps**:
1. Sync the project to pick up the updated playbook.
2. Launch the "Count Managed Nodes" job template — no extra vars.
3. In the job log, verify the new "Host Metrics" section appears between the disabled
   count line and the closing separator:
   ```
   Host Metrics (from host_metrics endpoint)
   Hosts in Metrics    : <number>
   Automated (≥1 run)  : <number>
   Never Automated     : <number>
   Most Recent Run     : <ISO timestamp>
   ```
4. After the job completes, pull the latest CSV from the GitHub `output/` directory
   and verify the five new columns are present with non-empty values.

**Expected result**: Job succeeds; metrics section populated; CSV has 11 columns.

---

## Scenario 2: Metrics Not Available (Ansible Tower)

**Setup**: Ansible Tower instance (no host_metrics endpoint).

**Steps**:
1. Launch the job template against the Tower instance.
2. In the job log, verify the metrics section shows:
   ```
   Host Metrics        : Not available on this platform
   ```
3. Verify the job exits with status **Successful** (not Failed).
4. Pull the CSV and verify the first 6 columns are populated; the 5 new columns
   are empty.

**Expected result**: Job succeeds; graceful degradation message in log; CSV schema consistent.

---

## Scenario 3: Permission Denied (403)

**Setup**: AAP instance with a credential that lacks System Auditor or higher role.

**Steps**:
1. Temporarily use a restricted credential.
2. Launch the job — verify it succeeds with:
   ```
   Host Metrics        : Not available (permission denied)
   ```

**Expected result**: Job succeeds with warning; no failure.

---

## Verification Checklist

- [ ] Metrics section appears in job log on AAP 2.x instance
- [ ] Job succeeds without error on Tower (404 handled gracefully)
- [ ] Job succeeds without error when credential lacks metrics permission (403)
- [ ] CSV in GitHub output/ has 11 columns with correct header
- [ ] CSV metrics columns are empty (not missing) when metrics unavailable
- [ ] Job run time increase is ≤5 seconds compared to pre-metrics baseline
