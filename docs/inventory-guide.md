# 📋 AAP Inventory Management Guide

A plain-language guide to managing inventories, hostnames, and managed node
counts in Ansible Automation Platform (AAP). Written for anyone who touches
AAP — you don't need to be an API expert.

---

## Table of Contents

1. [Why This Matters](#why-this-matters)
2. [Hostname Conventions](#hostname-conventions)
3. [Don't Put localhost in Your Inventory](#dont-put-localhost-in-your-inventory)
4. [Inventory Organization](#inventory-organization)
5. [Dynamic Inventory Over Static](#dynamic-inventory-over-static)
6. [Removing Hosts from an Inventory](#-removing-hosts-from-an-inventory)
7. [Removing Hosts from Host Metrics](#-removing-hosts-from-host-metrics)
8. [Common Pitfalls](#-common-pitfalls)

---

## Why This Matters

Every unique hostname in your AAP inventories counts as a **managed node**
against your Red Hat subscription. More nodes = higher cost. If the same
server shows up under three different names, AAP counts it as three nodes —
even though it's one machine.

Good inventory hygiene keeps your node count accurate, your subscription
costs predictable, and your automation reliable.

---

## Hostname Conventions

### ✅ Use FQDNs as inventory hostnames

The fully qualified domain name (e.g. `web-prod-01.example.com`) is the
most portable, readable, and deduplicate-friendly choice.

### ✅ Use `ansible_host` for the connection address

If a server's FQDN doesn't resolve directly (or you need to connect via IP),
set `ansible_host` as a variable — don't replace the hostname with an IP.

### ❌ Don't use bare IPs or short names as the primary hostname

They cause deduplication failures and inflate your managed node count.

### Example: The Wrong Way vs The Right Way

**Wrong — 3 managed nodes counted for 1 server:**

```ini
# inventory.ini
10.0.1.50
web-prod-01
web-prod-01.example.com
```

AAP sees three different hostnames. That's three nodes on your subscription
bill, even though they're all the same machine.

**Right — 1 managed node counted:**

```ini
# inventory.ini
web-prod-01.example.com ansible_host=10.0.1.50
```

One hostname, one node. The `ansible_host` variable tells Ansible where to
connect without creating extra entries.

---

## Don't Put localhost in Your Inventory

This is a common mistake. Ansible has a **built-in implicit localhost** —
you never need to add it to an inventory file.

### How it works

When a playbook says `hosts: localhost` with `connection: local`, Ansible
uses the implicit localhost automatically. No inventory entry required.

Here's a real example from this repo (`playbooks/count_managed_nodes.yml`):

```yaml
- name: Count Managed Nodes in Ansible Automation Platform
  hosts: localhost
  gather_facts: false
  connection: local

  vars:
    controller_host: "{{ lookup('env', 'CONTROLLER_HOST') }}"

  tasks:
    - name: Query the controller API
      ansible.builtin.uri:
        url: "{{ controller_host }}/api/v2/hosts/"
        method: GET
        # ...
```

This playbook runs on `localhost` with no inventory entry for it. The
implicit localhost handles everything.

### Why adding localhost to an inventory causes problems

| Problem | What happens |
|---------|-------------|
| 🎫 **Counts as a managed node** | If any job targets it, it burns a node slot on your subscription |
| ⚠️ **Variable conflicts** | The implicit localhost sets `ansible_connection: local` and `ansible_python_interpreter` automatically. An explicit entry can override these with wrong values |
| 🔀 **Duplication risk** | If multiple inventories each define localhost, you get variable precedence fights between them |

### What to do instead

Use `hosts: localhost` and `connection: local` in the play definition.
Put any variables you need in the play's `vars:` block — not in an
inventory file:

```yaml
- name: My local automation task
  hosts: localhost
  connection: local
  gather_facts: false

  vars:
    my_api_url: "https://api.example.com"

  tasks:
    - name: Call an API
      ansible.builtin.uri:
        url: "{{ my_api_url }}/endpoint"
        method: GET
```

No inventory entry. No wasted managed node. No variable conflicts.

---

## Inventory Organization

### 🗂️ Separate inventories by environment

Create one inventory per environment — `dev`, `staging`, `prod`. This gives
you clean RBAC boundaries and independent sync schedules.

### 🔗 Use Constructed Inventory for merged views

Need to see "all databases across every environment" in one place? Don't
cram everything into one inventory. Create separate inventories per
environment, then use **Constructed Inventory** to build filtered views
across them.

### ⚠️ Smart Inventory is deprecated

If you're on AAP 2.4+, Smart Inventory is deprecated. Migrate existing
Smart Inventories to Constructed Inventory.

### 🏷️ Tag everything consistently

For multi-cloud or hybrid environments, enforce a minimum tag set on every
resource across all providers:

| Tag | Purpose |
|-----|---------|
| `environment` | prod / staging / dev |
| `team` | Owning team |
| `application` | App or service name |
| `cost-center` | Billing code |

Without consistent tags, Constructed Inventory filters have nothing to
work with.

---

## Dynamic Inventory Over Static

For anything running in the cloud, **use dynamic inventory sources** (AWS
EC2, Azure RM, GCP Compute). When an instance terminates, it disappears
from the inventory on the next sync — no stale entries inflating your
node count.

### Recommended sync intervals

| Environment type | Sync frequency |
|-----------------|----------------|
| Ephemeral (CI/CD, containers) | Every 5–15 minutes |
| Stable production | Every 30–60 minutes |
| Development / lab | On-demand or hourly |

You can also enable **"Update on launch"** on the inventory source so the
inventory always refreshes before a job runs.

---

## 🗑️ Removing Hosts from an Inventory

When a server is decommissioned, remove it from the AAP inventory so it
stops counting as a managed node.

### API endpoint

```
DELETE /api/v2/hosts/{id}/
```

Returns `204 No Content` on success.

### curl example

```bash
curl -k -X DELETE \
  -H "Authorization: Bearer <your-token>" \
  "https://aap.example.com/api/v2/hosts/42/"
```

### Ansible module example

```yaml
- name: Remove a host from the inventory
  ansible.platform.host:
    controller_host: "https://aap.example.com"
    controller_oauthtoken: "{{ my_token }}"
    validate_certs: false
    name: "web-prod-01.example.com"
    inventory: "production"
    state: absent
```

> **Note:** `ansible.platform.host` is the current module. The older
> `ansible.controller.host` still works but is considered legacy.

---

## 📊 Removing Hosts from Host Metrics

Removing a host from an inventory doesn't remove it from **Host Metrics**.
Host Metrics is a separate system that tracks every host AAP has ever
automated — it's what Red Hat uses for subscription counting.

If you decommission a server, you should clean it up in both places.

### Querying host metrics

```
GET /api/v2/host_metrics/
```

Returns a paginated list of all hosts AAP has tracked. Each entry has an
`id` you'll need for deletion.

### Soft delete (recommended for most cases)

```
DELETE /api/v2/host_metrics/{id}/
```

Marks the host as deleted. The record stays in the database but **stops
counting toward your subscription**. If automation accidentally runs
against that hostname later, the record is automatically restored.

```bash
curl -k -X DELETE \
  -H "Authorization: Bearer <your-token>" \
  "https://aap.example.com/api/v2/host_metrics/87/"
```

### Hard delete (permanent removal)

```
DELETE /api/v2/host_metrics/{id}/?soft_delete=false
```

Permanently removes the record from the database. No history remains.

```bash
curl -k -X DELETE \
  -H "Authorization: Bearer <your-token>" \
  "https://aap.example.com/api/v2/host_metrics/87/?soft_delete=false"
```

### When to use which

| | Soft Delete | Hard Delete |
|---|---|---|
| **Endpoint** | `DELETE /api/v2/host_metrics/{id}/` | `DELETE /api/v2/host_metrics/{id}/?soft_delete=false` |
| **Record preserved** | ✅ Yes (flagged as deleted) | ❌ No (permanently removed) |
| **Counts toward subscription** | ❌ No | ❌ No |
| **Auto-restores if host re-automated** | ✅ Yes | ❌ No (creates new entry) |
| **Auto-purge** | After 36 months | Immediate |
| **Best for** | Decommissioned servers, subscription cleanup | Lab/test teardown, data purge, privacy compliance |

**Use soft delete** when the server is gone but you want the safety net. If
someone runs a playbook against that hostname by mistake, the record comes
back automatically — no gap in tracking.

**Use hard delete** when the host will never come back and you want a clean
table. Common for test/lab environments, CI/CD runners, or when hostnames
contain information you're required to purge.

---

## 🚧 Common Pitfalls

| Pitfall | What goes wrong | Fix |
|---------|----------------|-----|
| Same host, multiple names | 1 server counted as 3 managed nodes | Use FQDN + `ansible_host` for connectivity |
| localhost in inventory | Wastes a managed node slot, causes variable conflicts | Use `hosts: localhost` + `connection: local` in the play — no inventory entry |
| Static inventory in the cloud | Terminated instances stay in inventory forever | Use dynamic inventory sources with automatic sync |
| No cleanup policy | Decommissioned hosts inflate subscription count | Soft-delete from host_metrics, remove from inventory |
| Variables in the inventory file | Hard to read, version, and debug | Move variables to `group_vars/` and `host_vars/` directories |
| Ignoring tags on cloud resources | Can't filter or group hosts meaningfully | Enforce `environment`, `team`, `application` tags on all resources |
