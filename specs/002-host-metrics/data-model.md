# Data Model: Host Metrics Integration

**Branch**: `002-host-metrics` | **Date**: 2026-04-17

---

## Entities

### HostMetricsSummary

Aggregated metrics derived from the AAP `host_metrics` API endpoint.
Computed from multiple filtered API calls and stored as a single Ansible fact
(`host_metrics_summary`).

| Field                  | Type    | Source                                              | When Unavailable |
|------------------------|---------|-----------------------------------------------------|------------------|
| `available`            | bool    | Set based on HTTP status of metrics probe           | `false`          |
| `total_in_metrics`     | int     | `.count` from `GET /host_metrics/?page_size=1`      | `0`              |
| `automated`            | int     | `.count` from `GET /host_metrics/?automated_counter__gt=0&page_size=1` | `0` |
| `not_automated`        | int     | `total_in_metrics - automated`                      | `0`              |
| `most_recent_automation` | string | `results[0].last_automation` from `?ordering=-last_automation&page_size=1` | `''` |

---

### EnhancedCSVRow

Extends the existing CSV row (from feature 001) with host metrics fields.

| Column                    | Type   | Source                              |
|---------------------------|--------|-------------------------------------|
| `timestamp`               | string | Existing (run_timestamp)            |
| `controller_url`          | string | Existing                            |
| `total_count`             | int    | Existing                            |
| `enabled_count`           | int    | Existing                            |
| `disabled_count`          | int    | Existing                            |
| `job_id`                  | string | Existing                            |
| `metrics_available`       | bool   | `host_metrics_summary.available`    |
| `hosts_in_metrics`        | int    | `host_metrics_summary.total_in_metrics` |
| `hosts_automated`         | int    | `host_metrics_summary.automated`    |
| `hosts_not_automated`     | int    | `host_metrics_summary.not_automated` |
| `most_recent_automation`  | string | `host_metrics_summary.most_recent_automation` |

---

## API Response Shape (host_metrics endpoint)

### Probe call (page_size=1)

```json
{
  "count": 142,
  "next": "https://aap.example.com/api/controller/v2/host_metrics/?page=2&page_size=1",
  "previous": null,
  "results": [
    {
      "id": 1,
      "hostname": "web01.example.com",
      "first_automation": "2025-01-15T08:00:00Z",
      "last_automation": "2026-04-17T18:30:00Z",
      "automated_counter": 312,
      "deleted_hosts_count": 0,
      "used_in_inventories": 2
    }
  ]
}
```

### Unavailable (404 response)

```json
{"detail": "Not found."}
```

### Permission denied (403 response)

```json
{"detail": "You do not have permission to perform this action."}
```

---

## Notes

- `automated_counter__gt=0` filter relies on standard AAP DRF field lookup syntax.
  If the filter is unsupported, the fallback is to use `total_in_metrics` from the
  unfiltered count and accept that `hosts_automated` / `hosts_not_automated` will
  not be available (written as empty in CSV).
- Field names (`automated_counter`, `first_automation`) should be accessed with
  `| default(0, true)` guards to handle version-specific schema differences.
