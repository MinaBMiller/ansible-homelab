# Ansible Homelab

A collection of Ansible playbooks for provisioning and configuring a self-hosted homelab environment. Built to develop hands-on sysadmin and networking skills.

## Stack

- **Ansible** — configuration management and automation
- **UTM** — VM hypervisor on Apple Silicon Mac
- **Ubuntu Server** — target VMs

## Project Structure

```
ansible-homelab/
├── ansible.cfg          # Ansible configuration (inventory path, SSH defaults)
├── inventory/
│   ├── hosts.ini        # Target machines and groups
│   └── group_vars/      # Variables scoped to host groups
├── playbooks/           # Top-level playbooks (what gets run)
└── roles/               # Reusable task bundles (the actual work)
```

## Concepts Covered

- Inventory management and host grouping
- Playbook structure and task execution
- Roles and reusable configuration
- SSH key-based authentication
- System hardening (users, firewall, SSH config)
- Service installation and configuration

## Prerequisites

- Ansible installed on your control machine (`brew install ansible`)
- A Linux VM reachable over SSH (this project uses UTM on macOS)
- SSH key pair for passwordless auth to target VMs

## Getting Started

1. Create an Ubuntu Server VM in UTM
2. Update `inventory/hosts.ini` with the VM's IP address
3. Set up SSH key auth to the VM (see playbooks/bootstrap.yml)
4. Run your first playbook:

```bash
ansible-playbook playbooks/base.yml
```

## Why Ansible?

Ansible lets you define your infrastructure as code — instead of manually SSHing into servers and running commands, you write repeatable, readable automation. This means your setup is documented, version-controlled, and reproducible.
