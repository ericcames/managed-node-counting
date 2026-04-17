# Contract: count_managed_nodes Playbook Interface

**Branch**: `001-ansible-node-count` | **Date**: 2026-04-17

This document defines the interface contract for the `count_managed_nodes.yml`
playbook — what inputs it requires, what it produces, and what it guarantees.

---

## Inputs (Required)

All inputs are injected via a Custom Credential Type. No extra vars are permitted.

### Environment Variables (via Custom Credential Type)

| Variable                  | Required | Description                                    |
|---------------------------|----------|------------------------------------------------|
| `CONTROLLER_HOST`         | Yes      | FQDN or IP of the AAP/Tower controller         |
| `CONTROLLER_OAUTH_TOKEN`  | Yes*     | OAuth2 token with read access to Hosts API     |
| `CONTROLLER_USERNAME`     | No*      | Basic auth username (alternative to token)     |
| `CONTROLLER_PASSWORD`     | No*      | Basic auth password (alternative to token)     |

\* `CONTROLLER_OAUTH_TOKEN` is preferred. If not present,
`CONTROLLER_USERNAME` + `CONTROLLER_PASSWORD` must both be set.

### Custom Credential Type Definition

Input configuration (for AAP/Tower UI):

```yaml
fields:
  - id: controller_host
    type: string
    label: "Controller Host"
    help_text: "FQDN or IP of the AAP/Tower controller (e.g. https://aap.example.com)"
  - id: controller_oauth_token
    type: string
    label: "Controller OAuth2 Token"
    secret: true
    help_text: "Personal access token with read access to the Hosts API"
required:
  - controller_host
  - controller_oauth_token
```

Injector configuration:

```yaml
env:
  CONTROLLER_HOST: "{{ controller_host }}"
  CONTROLLER_OAUTH_TOKEN: "{{ controller_oauth_token }}"
```

---

## Outputs

### Job Log (always)

The playbook writes to stdout (job log) in the following format:

```
==============================================
  Managed Node Count Report
  Controller: https://aap.example.com
  Timestamp:  2026-04-17T14:32:00Z
  Job ID:     42
----------------------------------------------
  Total Managed Nodes:    1427
  Enabled Nodes:          1389
  Disabled Nodes:            38
==============================================
```

### CSV File (on success)

Written to `output/managed_node_count.csv` relative to the project directory:

```
timestamp,controller_url,total_count,enabled_count,disabled_count,job_id
2026-04-17T14:32:00Z,https://aap.example.com,1427,1389,38,42
```

The file is **overwritten** on each successful run (not appended).

---

## Guarantees

| Guarantee                          | Condition                              |
|------------------------------------|----------------------------------------|
| First task is always an assert     | Unconditional                          |
| No tasks run if assert fails       | Credential check fails fast            |
| Job log contains count on success  | API query succeeded                    |
| CSV written on success             | Output directory exists and is writable|
| Idempotent count result            | Same estate state → same count         |
| No extra vars required             | Unconditional                          |

---

## Error Conditions

| Condition                          | Behaviour                                        |
|------------------------------------|--------------------------------------------------|
| `CONTROLLER_HOST` not set          | Assert fails, job errors with descriptive message|
| `CONTROLLER_OAUTH_TOKEN` not set   | Assert fails, job errors with descriptive message|
| API returns non-200 status         | Task fails with HTTP status code in error output |
| API unreachable (network error)    | `uri` module fails with connection error         |
| Output directory not writable      | `copy`/`template` task fails with permission error|
| Zero managed nodes                 | Success; count of 0 written to log and CSV       |

---

## Compatibility

| Platform              | Version         | Status              |
|-----------------------|-----------------|---------------------|
| Ansible Automation Platform | 2.4 (ansible-core 2.15) | Supported |
| Ansible Automation Platform | 2.5 (ansible-core 2.16) | Supported |
| Ansible Automation Platform | 2.6 (ansible-core 2.16) | Supported |
| Ansible Tower         | 3.8.x           | Best-effort (legacy)|
| Ansible Tower         | < 3.8           | Not supported        |

---

## Collections Used

| Collection            | Source                        | Use                           |
|-----------------------|-------------------------------|-------------------------------|
| `ansible.builtin`     | Included with ansible-core    | `uri`, `assert`, `copy`, `set_fact`, `debug` |

No additional collection installations are required.
