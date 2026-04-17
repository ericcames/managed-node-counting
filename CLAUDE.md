# managed_node_counting Development Guidelines

Auto-generated from all feature plans. Last updated: 2026-04-17

## Active Technologies
- YAML (Ansible playbook), ansible-core 2.15 minimum (AAP 2.4 baseline) + `ansible.builtin` only — `uri`, `set_fact`, `debug`, `copy` (no additional collections) (002-host-metrics)
- GitHub repository `output/` directory via Contents API (same as feature 001) (002-host-metrics)
- GitHub repository `output/` directory via Contents API (unchanged) (003-job-events-hosts)

- YAML (Ansible playbook), ansible-core 2.15 minimum (AAP 2.4 baseline) + `ansible.builtin` (certified, no additional installs) (001-ansible-node-count)

## Project Structure

```text
src/
tests/
```

## Commands

# Add commands for YAML (Ansible playbook), ansible-core 2.15 minimum (AAP 2.4 baseline)

## Code Style

YAML (Ansible playbook), ansible-core 2.15 minimum (AAP 2.4 baseline): Follow standard conventions

## Recent Changes
- 003-job-events-hosts: Added YAML (Ansible playbook), ansible-core 2.15 minimum (AAP 2.4 baseline) + `ansible.builtin` only — `uri`, `set_fact`, `debug`, `copy` (no additional collections)
- 002-host-metrics: Added YAML (Ansible playbook), ansible-core 2.15 minimum (AAP 2.4 baseline) + `ansible.builtin` only — `uri`, `set_fact`, `debug`, `copy` (no additional collections)

- 001-ansible-node-count: Added YAML (Ansible playbook), ansible-core 2.15 minimum (AAP 2.4 baseline) + `ansible.builtin` (certified, no additional installs)

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
