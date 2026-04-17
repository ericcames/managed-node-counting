# Data Model: Job Events Per-Host Enrichment

**Branch**: `003-job-events-hosts` | **Date**: 2026-04-17

---

## Entities

### RecentJobList

List of up to 10 most recent completed job IDs, used to scope the event query.
The current running job ID is excluded.

| Field        | Type   | Source                                                          |
|--------------|--------|-----------------------------------------------------------------|
| `job_ids`    | list   | `GET /jobs/?status__in=successful,failed&order_by=-id&page_size=10` → `results[*].id` |
| `available`  | bool   | True when at least one job ID was found                        |

---

### HostActivitySummary

Per-host rollup of most recent automation event across the last 10 jobs.
Stored as the Ansible fact `host_activity_list` (list of dicts).

| Field         | Type   | Source                                                       | When Unavailable |
|---------------|--------|--------------------------------------------------------------|------------------|
| `hostname`    | string | `host_name` field from event, lowercased                     | —                |
| `last_status` | string | Mapped from `event` field: ok / failed / unreachable / skipped | —             |
| `last_seen`   | string | `created` timestamp of most recent event for this host       | —                |

---

### JobEventAggregates

Two scalar counts derived from `host_activity_list`. Appended to `count_result`
and the CSV row.

| Field                        | Type | Source                                                          | When Unavailable |
|------------------------------|------|-----------------------------------------------------------------|------------------|
| `hosts_with_recent_failures` | int  | Count of hosts in `host_activity_list` where `last_status` in `[failed, unreachable]` | `''` |
| `hosts_with_recent_success`  | int  | Count of hosts where `last_status` in `[ok]`                  | `''`             |
| `job_events_available`       | bool | True when event data was successfully retrieved and non-empty  | `false`          |

---

## Event Type → Status Mapping

| `event` field value    | Mapped `last_status` | Counts as failure? |
|------------------------|----------------------|--------------------|
| `runner_on_ok`         | `ok`                 | No                 |
| `runner_on_changed`    | `ok`                 | No                 |
| `runner_on_skipped`    | `skipped`            | No                 |
| `runner_on_failed`     | `failed`             | Yes                |
| `runner_on_unreachable`| `unreachable`        | Yes                |
| other / unknown        | `unknown`            | No                 |

---

## API Response Shape

### Jobs query (to get recent job IDs)

```json
{
  "count": 143,
  "results": [
    {"id": 143, "status": "successful", "finished": "2026-04-17T21:40:57Z"},
    {"id": 141, "status": "successful", "finished": "2026-04-17T21:35:00Z"}
  ]
}
```

### Job events query

```json
{
  "count": 48,
  "next": null,
  "results": [
    {
      "id": 5821,
      "job": 143,
      "event": "runner_on_ok",
      "host_name": "web01.example.com",
      "task": "Gather facts",
      "created": "2026-04-17T21:40:45Z",
      "failed": false,
      "changed": false
    },
    {
      "id": 5822,
      "job": 141,
      "event": "runner_on_failed",
      "host_name": "db02.example.com",
      "task": "Install packages",
      "created": "2026-04-17T21:35:10Z",
      "failed": true,
      "changed": false
    }
  ]
}
```

### Enhanced CSV Row (13 columns)

```
timestamp,controller_url,total_count,enabled_count,disabled_count,job_id,
metrics_available,hosts_in_metrics,hosts_automated,hosts_not_automated,
most_recent_automation,hosts_with_recent_failures,hosts_with_recent_success
```
