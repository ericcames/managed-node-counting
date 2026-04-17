# Contract: Playbook Interface — Job Events Per-Host Enrichment

**Branch**: `003-job-events-hosts` | **Date**: 2026-04-17

---

## Credential Types (unchanged from features 001 + 002)

No new credential type fields or job template changes required.

---

## New Ansible Facts Set by This Feature

| Fact                    | Type | Description                                              |
|-------------------------|------|----------------------------------------------------------|
| `recent_job_ids`        | list | Up to 10 most recent completed job IDs (current excluded)|
| `job_events_available`  | bool | True when events were retrieved and contain host data    |
| `host_activity_list`    | list | Per-host dicts: hostname, last_status, last_seen         |
| `job_event_aggregates`  | dict | hosts_with_recent_failures, hosts_with_recent_success    |

---

## Job Log Output Contract

### When job event data IS available

The existing report is extended with a new section after the Host Metrics block:

```
==============================================
  Managed Node Count Report
  ...
----------------------------------------------
  Host Metrics (from host_metrics endpoint)
  ...
----------------------------------------------
  Recent Host Activity (last 10 jobs)
  web01.example.com   : ok          (2026-04-17T21:40:45Z)
  db02.example.com    : failed      (2026-04-17T21:35:10Z)
  app03.example.com   : unreachable (2026-04-17T21:30:00Z)
  Hosts with failures : 2
  Hosts successful    : 1
==============================================
```

### When no job event data is available

```
----------------------------------------------
  Recent Host Activity : No recent job event data available
==============================================
```

---

## CSV Schema Contract (13 columns, append-only)

Header row:
```
timestamp,controller_url,total_count,enabled_count,disabled_count,job_id,metrics_available,hosts_in_metrics,hosts_automated,hosts_not_automated,most_recent_automation,hosts_with_recent_failures,hosts_with_recent_success
```

**Data available**:
```
2026-04-17T21:40:57Z,https://aap.example.com,12,12,0,146,True,5,5,0,,2,1
```

**Data unavailable** (empty trailing fields):
```
2026-04-17T21:40:57Z,https://aap.example.com,12,12,0,146,True,5,5,0,,,
```

---

## Backward Compatibility

- Existing 11 columns (features 001 + 002) remain in the same positions.
- 2 new columns appended to the right — existing parsers continue to work.
- No changes to credential type definitions or job template configuration.
