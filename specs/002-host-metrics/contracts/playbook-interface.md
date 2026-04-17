# Contract: Playbook Interface — Host Metrics Extension

**Branch**: `002-host-metrics` | **Date**: 2026-04-17

---

## Credential Types (unchanged from feature 001)

No new credential type fields are required. The existing credentials inject:

- `CONTROLLER_HOST` — AAP/Tower base URL
- `CONTROLLER_OAUTH_TOKEN` — OAuth2 token (preferred auth)
- `CONTROLLER_USERNAME` / `CONTROLLER_PASSWORD` — basic auth fallback
- `GITHUB_TOKEN`, `GITHUB_REPO_OWNER`, `GITHUB_REPO_NAME`, `GITHUB_REPO_BRANCH`

---

## New Ansible Facts Set by This Feature

| Fact                    | Type | Description                                          |
|-------------------------|------|------------------------------------------------------|
| `host_metrics_summary`  | dict | Aggregate metrics (see data-model.md for schema)     |
| `host_metrics_available`| bool | True when endpoint returned HTTP 200                 |

---

## Job Log Output Contract

### When host metrics ARE available

The existing "Managed Node Count Report" block is extended with a metrics section:

```
==============================================
  Managed Node Count Report
  Controller : https://aap.example.com
  API Base   : https://aap.example.com/api/controller/v2
  Timestamp  : 2026-04-17T18:30:00Z
  Job ID     : 142
----------------------------------------------
  Total Managed Nodes : 95
  Enabled Nodes       : 90
  Disabled Nodes      : 5
----------------------------------------------
  Host Metrics (from host_metrics endpoint)
  Hosts in Metrics    : 142
  Automated (≥1 run)  : 130
  Never Automated     : 12
  Most Recent Run     : 2026-04-17T18:30:00Z
==============================================
```

### When host metrics are NOT available (404)

```
==============================================
  Managed Node Count Report
  ...
----------------------------------------------
  Host Metrics        : Not available on this platform
==============================================
```

### When host metrics return 403

```
----------------------------------------------
  Host Metrics        : Not available (permission denied)
                        Ensure credential has System Auditor role.
==============================================
```

---

## CSV Schema Contract

Header row (column order is fixed and append-only):

```
timestamp,controller_url,total_count,enabled_count,disabled_count,job_id,metrics_available,hosts_in_metrics,hosts_automated,hosts_not_automated,most_recent_automation
```

Data row examples:

**Metrics available**:
```
2026-04-17T18:30:00Z,https://aap.example.com,95,90,5,142,True,142,130,12,2026-04-17T18:30:00Z
```

**Metrics unavailable** (empty trailing fields):
```
2026-04-17T18:30:00Z,https://aap.example.com,95,90,5,142,False,,,, 
```

---

## Backward Compatibility

- Existing CSV columns remain in the same positions — parsers reading only the first
  6 columns continue to work without modification.
- The job log report block heading and structure are preserved; the metrics section
  is appended after the existing disabled count line.
- No changes to the credential type definitions.
- No changes to the job template configuration (no new extra vars, no new credential
  attachments required beyond what feature 001 already requires).
