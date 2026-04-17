# Known Limitations

This document describes the known constraints and boundaries of the
`count_managed_nodes.yml` playbook.

---

## 1. Deduplication Is Name-Only

The playbook deduplicates hosts by normalizing names to lowercase and removing
duplicates. It does **not** resolve whether an IP address and a fully-qualified
domain name refer to the same physical or virtual machine.

**Example**: A host registered as both `192.168.1.10` and `server1.example.com`
will be counted as **two** managed nodes, even if they are the same machine.

**Workaround**: Ensure hosts are registered consistently (all by FQDN or all by
IP) within your AAP/Tower inventories to get the most accurate count.

---

## 2. CSV Is Not Automatically Committed to Git

The CSV report is written to `output/managed_node_count.csv` on the execution
node's working copy of the project. It is **not** automatically committed or
pushed to the SCM repository.

**To persist the CSV**:
- Retrieve the file from the execution node after the job completes, or
- Manually commit `output/managed_node_count.csv` to your git repository after
  each run.

Note: `output/managed_node_count.csv` is listed in `.gitignore` by default.
Remove the entry from `.gitignore` if you want to track it in version control.

---

## 3. Count Reflects State at Execution Time

The playbook queries the controller API at the moment the job runs. The count
is a point-in-time snapshot — it does not provide historical trending or
track changes over time.

**To build a history**: Run the job on a schedule and commit the CSV output
after each run to maintain a timestamped record.

---

## 4. Ansible Tower Is Legacy (Best-Effort Support)

Ansible Tower is no longer listed in Red Hat's current Ansible Automation
Platform support lifecycle. Tower support in this playbook is best-effort only.

- Tower 3.8.x is tested against `/api/v2/` endpoints
- Tower versions below 3.8 are not supported
- Red Hat recommends migrating to AAP 2.x

---

## 5. TLS Certificate Validation Is Disabled

The playbook sets `validate_certs: false` on all API calls to support
controllers with self-signed certificates. This is common in lab and workshop
environments.

**Security note**: In production environments with valid TLS certificates,
change `validate_certs: false` to `validate_certs: true` in
`playbooks/count_managed_nodes.yml`.

---

## 6. Credential Must Have API Read Access

The credential attached to the job template must have read access to the
controller's `/hosts/` API endpoint. An account with at least **System
Auditor** or **Org Admin** role is required.

Jobs will fail at the API query step if the credential has insufficient
permissions, even if the assert task passes.

---

## 7. Performance on Large Estates

The playbook fetches all hosts in pages of 200. For very large estates
(tens of thousands of nodes), execution time grows linearly with the number
of hosts. The playbook targets completion within 60 seconds for typical
estate sizes but may exceed this for unusually large inventories.

---

## 8. AAP 2.x vs Tower API Path

The playbook auto-detects whether it is running against AAP 2.x
(`/api/controller/v2/`) or Ansible Tower / AAP 1.x (`/api/v2/`) by
probing the `/api/` gateway endpoint. If auto-detection fails due to an
unexpected API response structure, the playbook falls back to `/api/v2/`.

---

## 9. Enabled/Disabled Count Edge Case

A host that appears in multiple inventories — once with `enabled: true` and
once with `enabled: false` — will be counted as **enabled** in the enabled
count after deduplication (any enabled instance marks the name as enabled).
This matches the most permissive interpretation of node state.
