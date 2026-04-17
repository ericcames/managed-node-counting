# Quickstart: Managed Node Count Playbook

**Branch**: `001-ansible-node-count` | **Date**: 2026-04-17

Validate end-to-end functionality after implementation.

---

## Prerequisites

- **Test instance details recorded** in `docs/dev-environment.md` (see task T000):
  - Controller URL (AAP 2.4/2.5/2.6 or Ansible Tower 3.8.x)
  - Username with Admin or Org Admin permissions
  - Password for that account
- The SCM project synced and the playbook present at `playbooks/count_managed_nodes.yml`
- At least one managed node registered with the controller (or accept count=0)

---

## Validation Steps

### Step 1: Create the Custom Credential Type

In the AAP/Tower UI → Credential Types → Add:

- **Name**: `AAP Node Count Credential`
- **Input Configuration** (YAML):
  ```yaml
  fields:
    - id: controller_host
      type: string
      label: "Controller Host"
    - id: controller_oauth_token
      type: string
      label: "Controller OAuth2 Token"
      secret: true
  required:
    - controller_host
    - controller_oauth_token
  ```
- **Injector Configuration** (YAML):
  ```yaml
  env:
    CONTROLLER_HOST: "{{ controller_host }}"
    CONTROLLER_OAUTH_TOKEN: "{{ controller_oauth_token }}"
  ```

### Step 2: Create a Credential

In the UI → Credentials → Add:

- **Credential Type**: `AAP Node Count Credential`
- **Controller Host**: `https://<your-controller-fqdn>`
- **Controller OAuth2 Token**: generate via Profile → Tokens in the UI

### Step 3: Create a Job Template

In the UI → Templates → Add → Job Template:

- **Name**: `Count Managed Nodes`
- **Inventory**: Any (the playbook targets localhost)
- **Project**: Your SCM project containing this repository
- **Playbook**: `playbooks/count_managed_nodes.yml`
- **Credentials**: Add the `AAP Node Count Credential` created above
- **Extra Variables**: Leave empty (none required)

### Step 4: Launch the Job

Click **Launch** on the job template. Observe:

- [ ] First task is "Assert required credentials are present" — PASSED
- [ ] Job log shows the "Managed Node Count Report" block
- [ ] `total_count` is a non-negative integer
- [ ] `enabled_count` + `disabled_count` = `total_count`
- [ ] Job completes with status: **Successful**

### Step 5: Verify CSV Output

After the job completes, check `output/managed_node_count.csv` exists in the
project's working directory. If you have access to the execution node or have
configured artifact retrieval, verify:

- [ ] CSV has the correct header row
- [ ] Data row timestamp matches the job run time
- [ ] `controller_url` matches your controller FQDN
- [ ] Node counts match the job log output

---

## Failure Scenario Validation

### Test: Missing Credential

1. Create a second job template with NO credential attached.
2. Launch the job.
3. Expected: First assert task fails with message identifying the missing
   environment variable. No API calls are made.

### Test: API Unreachable

1. Set `CONTROLLER_HOST` to an invalid URL in the credential.
2. Launch the job.
3. Expected: The `uri` task fails with a connection error. Job status: Failed.
