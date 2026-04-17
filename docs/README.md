# Managed Node Count Playbook

Count unique managed nodes registered in Ansible Automation Platform (AAP) or
Ansible Tower via the controller REST API. Results are written to the job log
and to `output/managed_node_count.csv`.

**Supported platforms**: AAP 2.4, 2.5, 2.6 | Ansible Tower 3.8.x (legacy)
**Content**: `ansible.builtin` only — no additional collection installs required
**Execution**: GUI-driven — no extra variables needed

See [Known Limitations](limitations.md) before deploying to production.

---

## Table of Contents

1. [Setup on Ansible Automation Platform (AAP 2.x)](#setup-on-ansible-automation-platform-aap-2x)
2. [Setup on Ansible Tower (Legacy)](#setup-on-ansible-tower-legacy)
3. [Running the Playbook](#running-the-playbook)
4. [Understanding the Output](#understanding-the-output)
5. [Known Limitations](#known-limitations)

---

## Setup on Ansible Automation Platform (AAP 2.x)

### Prerequisites

- AAP 2.4, 2.5, or 2.6 with web UI access
- An account with **Organization Admin** or **System Administrator** role
- This repository added as an SCM project in AAP
- An OAuth2 token or username/password for a read-capable account

---

### Step 1: Create the Custom Credential Type

1. Log in to the AAP web UI.
2. Navigate to **Automation Execution → Infrastructure → Credential Types**.
3. Click **Create credential type**.
4. Set the **Name**: `AAP Node Count Credential`
5. Paste the following into **Input configuration**:

```yaml
fields:
  - id: controller_host
    type: string
    label: "Controller Host"
    help_text: "Full HTTPS URL of the controller. Example: https://aap.example.com"
  - id: controller_oauth_token
    type: string
    label: "Controller OAuth2 Token"
    secret: true
    help_text: "OAuth2 token with read access. Leave blank to use username/password."
  - id: controller_username
    type: string
    label: "Controller Username"
    help_text: "Username for basic auth (used only if OAuth2 token is blank)."
  - id: controller_password
    type: string
    label: "Controller Password"
    secret: true
    help_text: "Password for basic auth (used only if OAuth2 token is blank)."
required:
  - controller_host
```

6. Paste the following into **Injector configuration**:

```yaml
env:
  CONTROLLER_HOST: "{{ controller_host }}"
  CONTROLLER_OAUTH_TOKEN: "{{ controller_oauth_token }}"
  CONTROLLER_USERNAME: "{{ controller_username }}"
  CONTROLLER_PASSWORD: "{{ controller_password }}"
```

7. Click **Save**.

---

### Step 2: Generate an OAuth2 Token (Recommended)

1. Click your username in the top-right corner → **User details**.
2. Go to the **Tokens** tab → **Create token**.
3. Set **Scope** to `Read` and click **Save**.
4. Copy the token — it will only be shown once.

---

### Step 3: Create a Credential

1. Navigate to **Automation Execution → Infrastructure → Credentials**.
2. Click **Create credential**.
3. Set:
   - **Name**: `AAP Node Count - <your-controller-name>`
   - **Credential Type**: `AAP Node Count Credential`
   - **Controller Host**: `https://<your-aap-fqdn>` (no trailing slash)
   - **Controller OAuth2 Token**: paste the token from Step 2
4. Click **Save**.

---

### Step 4: Add the SCM Project

1. Navigate to **Automation Execution → Infrastructure → Projects**.
2. Click **Create project**.
3. Set:
   - **Name**: `Managed Node Count`
   - **Source Control Type**: `Git`
   - **Source Control URL**: URL of this repository
   - **Source Control Branch**: your branch (e.g. `main` or `001-ansible-node-count`)
4. Click **Save** and wait for the project sync to complete (green icon).

---

### Step 5: Create the Job Template

1. Navigate to **Automation Execution → Templates**.
2. Click **Create template → Create job template**.
3. Set:
   - **Name**: `Count Managed Nodes`
   - **Job Type**: `Run`
   - **Inventory**: Select any inventory (the playbook targets `localhost`)
   - **Project**: `Managed Node Count`
   - **Playbook**: `playbooks/count_managed_nodes.yml`
   - **Credentials**: Add the `AAP Node Count Credential` created in Step 3
   - **Extra Variables**: Leave **empty** — no extra variables are required or permitted
4. Click **Save**.

---

## Setup on Ansible Tower (Legacy)

> ⚠️ **Ansible Tower is end-of-life.** Red Hat recommends migrating to AAP 2.x.
> Tower support is best-effort. See [limitations.md](limitations.md) for details.

The setup process is identical to AAP 2.x with the following UI differences:

| Step | AAP 2.x Path | Ansible Tower Path |
|------|--------------|--------------------|
| Credential Types | Automation Execution → Infrastructure → Credential Types | Settings → Credential Types |
| Credentials | Automation Execution → Infrastructure → Credentials | Resources → Credentials |
| Projects | Automation Execution → Infrastructure → Projects | Resources → Projects |
| Templates | Automation Execution → Templates | Resources → Templates |
| Tokens | User details → Tokens | User icon → Tokens |

The API path on Tower is `/api/v2/` — the playbook detects this automatically.

**Tower version requirement**: 3.8.x or later. Earlier versions are not supported.

---

## Running the Playbook

1. Open the **Count Managed Nodes** job template.
2. Click **Launch**.
3. Do not add any extra variables — the credential provides all required input.
4. Monitor the job output in the **Output** tab.

---

## Understanding the Output

### Job Log

A successful run prints:

```
==============================================
  Managed Node Count Report
  Controller : https://aap.example.com
  API Base   : https://aap.example.com/api/controller/v2
  Timestamp  : 2026-04-17T14:32:00Z
  Job ID     : 42
----------------------------------------------
  Total Managed Nodes : 1427
  Enabled Nodes       : 1389
  Disabled Nodes      : 38
----------------------------------------------
  Host Metrics (from host_metrics endpoint)
  Hosts in Metrics    : 1389
  Automated (>=1 run) : 1350
  Never Automated     : 39
  Most Recent Run     : 2026-04-17T14:30:00Z
----------------------------------------------
  Recent Host Activity (last 10 jobs)
  web01.example.com : ok (2026-04-17T14:30:00Z)
  db02.example.com  : failed (2026-04-17T14:29:00Z)
  Hosts with failures : 1
  Hosts successful    : 1
==============================================
NOTE: Deduplication is case-insensitive name normalization.
Hosts registered as both IP and FQDN count as separate nodes.
See docs/limitations.md for full details.
```

### CSV File

After a successful run, `output/managed_node_count.csv` contains 13 columns:

```
timestamp,controller_url,total_count,enabled_count,disabled_count,job_id,metrics_available,hosts_in_metrics,hosts_automated,hosts_not_automated,most_recent_automation,hosts_with_recent_failures,hosts_with_recent_success
2026-04-17T18:30:00Z,https://aap.example.com,95,90,5,142,True,142,130,12,2026-04-17T18:30:00Z,1,10
```

When the host metrics endpoint is not available (older Tower), columns 8–11 are
empty. When no job event history exists, columns 12–13 are empty:

```
2026-04-17T18:30:00Z,https://aap.example.com,95,90,5,142,False,,,,,, 
```

The file is **overwritten** on each run and also pushed to the GitHub repository
`output/` directory automatically. See [limitations.md](limitations.md).

---

### Recent Host Activity

On AAP 2.4 and later, the playbook queries the `job_events` endpoint across the
last 10 completed jobs to report per-host automation status:

| Field | Description |
|-------|-------------|
| `hosts_with_recent_failures` | Hosts whose last recorded event was `failed` or `unreachable` |
| `hosts_with_recent_success` | Hosts whose last recorded event was `ok` |

The "Recent Host Activity" section in the job log lists each host with its most
recent event status (`ok`, `failed`, `unreachable`, `skipped`) and the ISO
timestamp of that event.

**Graceful degradation**: If no completed jobs exist in AAP history, or if the
event endpoint returns no host-level events, the job log shows "No recent job
event data available" and the two CSV columns are empty. The job always succeeds.

**Requirements**: At least one prior completed job run (other than the count
playbook itself) must exist in AAP history. The count job's own events are
excluded from the query.

---

### Host Metrics

On AAP 2.4 and later, the playbook also queries the `host_metrics` endpoint to
report automation coverage:

| Field | Description |
|-------|-------------|
| `hosts_in_metrics` | Total hosts tracked by AAP's metrics system |
| `hosts_automated` | Hosts that have been automated at least once |
| `hosts_not_automated` | Hosts registered but never automated |
| `most_recent_automation` | ISO timestamp of the most recent automation run |

**Graceful degradation**: If the endpoint is unavailable (Ansible Tower or AAP 1.x),
the playbook logs "Not available on this platform" and continues without error.
The host metrics columns in the CSV will be empty.

**Permissions**: The same credential used for host counts also covers the
`host_metrics` endpoint. No additional credential configuration is needed.

---

## Known Limitations

| # | Limitation | Impact |
|---|-----------|--------|
| 1 | Deduplication is name-only (IP ≠ FQDN) | May overcount same host registered multiple ways |
| 2 | CSV not auto-committed to git | Manual retrieval required |
| 3 | Count is point-in-time | No historical trending |
| 4 | Ansible Tower is legacy | Best-effort support only |
| 5 | TLS validation disabled (`validate_certs: false`) | Security risk on public networks |
| 6 | Credential needs API read access | Job fails without System Auditor role |
| 7 | Performance degrades on very large estates | May exceed 60s for 10,000+ hosts |
| 8 | API path auto-detected; fallback to `/api/v2/` | May fail on unusual configurations |
| 9 | Enabled/disabled edge case with multi-inventory hosts | Any enabled instance marks name as enabled |
| 10 | Host metrics field names may vary by AAP version | `automated_counter` inferred — see limitations.md |
| 11 | Job event history may be purged by AAP retention | Activity section shows "No recent data" on active systems |

See [limitations.md](limitations.md) for full details on each item.
