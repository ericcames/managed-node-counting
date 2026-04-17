# Data Model: Ansible Managed Node Count Playbook

**Branch**: `001-ansible-node-count` | **Date**: 2026-04-17

---

## Entities

### HostCountResult

Represents the output of a single managed node count operation.

| Field            | Type     | Source                          | Notes                              |
|------------------|----------|---------------------------------|------------------------------------|
| `timestamp`      | datetime | Set at playbook execution start | ISO 8601, UTC                      |
| `controller_url` | string   | `CONTROLLER_HOST` env var       | FQDN or IP of AAP/Tower controller |
| `total_count`    | integer  | `/api/v2/hosts/` → `.count`     | All unique hosts in controller DB  |
| `enabled_count`  | integer  | `/api/v2/hosts/?enabled=true`   | Hosts with enabled=true            |
| `disabled_count` | integer  | Derived: total − enabled        | Hosts with enabled=false           |
| `job_id`         | string   | `AWX_JOB_ID` env var (AAP)      | AAP/Tower job ID for traceability  |

**Validation rules**:
- `total_count` MUST equal `enabled_count` + `disabled_count`
- `total_count` MUST be ≥ 0 (zero is valid — empty estate)
- `controller_url` MUST be non-empty (assert task enforces this)

---

### CSVRecord

The persisted form of a HostCountResult written to `output/managed_node_count.csv`.

**CSV Schema** (header row + one data row per run, file overwritten each run):

```
timestamp,controller_url,total_count,enabled_count,disabled_count,job_id
2026-04-17T14:32:00Z,https://aap.example.com,1427,1389,38,42
```

| Column           | Type    | Required | Notes                               |
|------------------|---------|----------|-------------------------------------|
| `timestamp`      | string  | Yes      | ISO 8601 UTC timestamp of the run   |
| `controller_url` | string  | Yes      | URL of the AAP/Tower controller     |
| `total_count`    | integer | Yes      | Total unique managed nodes          |
| `enabled_count`  | integer | Yes      | Enabled hosts only                  |
| `disabled_count` | integer | Yes      | Disabled hosts only                 |
| `job_id`         | string  | Yes      | Job ID or "N/A" if not available    |

---

### CredentialContext

Represents the credential variables injected by the Custom Credential Type.
These are environment variables — they have no persistent storage.

| Variable                | Required | Notes                                       |
|-------------------------|----------|---------------------------------------------|
| `CONTROLLER_HOST`       | Yes      | Controller FQDN or IP, no trailing slash    |
| `CONTROLLER_OAUTH_TOKEN`| Yes*     | OAuth2 token with Hosts API read access     |
| `CONTROLLER_USERNAME`   | No*      | Alternative: basic auth username            |
| `CONTROLLER_PASSWORD`   | No*      | Alternative: basic auth password            |

\* Either `CONTROLLER_OAUTH_TOKEN` OR `CONTROLLER_USERNAME`+`CONTROLLER_PASSWORD`
must be present. OAuth2 token is preferred.

---

## State Transitions

The playbook has a linear execution model — no state machine. Execution flow:

```
START
  └─► Assert credentials present
        ├── FAIL → Job fails with error message (no further tasks run)
        └── PASS
              └─► Query /api/v2/hosts/ for total count
                    ├── API error → Job fails with HTTP status + error body
                    └── SUCCESS
                          └─► Query enabled/disabled breakdown
                                └─► Write job log output
                                      └─► Write CSV to output/
                                            └─► END (success)
```

---

## File Layout

```
output/
└── managed_node_count.csv    # Written by playbook; overwritten each run

playbooks/
└── count_managed_nodes.yml   # Main playbook (references entities above)
```
