# Ansible Homelab

A collection of Ansible playbooks for provisioning and configuring a self-hosted homelab environment. Built to develop hands-on sysadmin and networking skills.

## Stack

- **Ansible** — configuration management and automation
- **UTM** — VM hypervisor on Apple Silicon Mac
- **Ubuntu Server** — target VMs

## Project Structure

```
ansible-homelab/
├── ansible.cfg              # Ansible configuration (inventory, SSH, roles path)
├── requirements.yml         # Ansible Galaxy collection dependencies
├── .env                     # Local environment variables (gitignored)
├── inventory/
│   ├── hosts.ini            # Your local inventory with VM IPs (gitignored)
│   ├── hosts.ini.example    # Template to copy for hosts.ini
│   └── group_vars/          # Variables scoped to host groups
├── playbooks/
│   ├── bootstrap.yml        # One-time setup (creates ansible user + SSH key)
│   └── base.yml             # Main playbook — runs all roles
└── roles/
    ├── common/              # apt updates, packages, UFW firewall, SSH hardening, timezone
    ├── users/               # user accounts, groups, SSH keys, sudo
    ├── networking/          # hostname, DNS, static IP, /etc/hosts
    ├── systemd/             # journal log limits, service management, custom units
    ├── monitoring/          # Node Exporter (port 9100), logwatch
    └── nginx/               # web server, reverse proxy, security headers, TLS-ready
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

1. Create an Ubuntu Server 24.04 ARM64 VM in UTM
2. Generate an SSH key if you don't have one: `ssh-keygen -t ed25519`
3. Copy the inventory template and set your VM's IP:
   ```bash
   cp inventory/hosts.ini.example inventory/hosts.ini
   # Edit hosts.ini with your VM's IP (find it with `ip a` on the VM)
   ```
4. Create a `.env` file with your SSH public key:
   ```bash
   echo "MINA_SSH_PUB_KEY=$(cat ~/.ssh/id_ed25519.pub)" > .env
   ```
5. Install required Ansible collections:
   ```bash
   ansible-galaxy collection install -r requirements.yml
   ```
6. Run the bootstrap playbook (one-time, sets up the `ansible` user):
   ```bash
   ansible-playbook playbooks/bootstrap.yml --ask-pass --ask-become-pass -u mina
   ```
7. Source your env and run the base playbook:
   ```bash
   source .env && export MINA_SSH_PUB_KEY
   ansible-playbook playbooks/base.yml
   ```

## Learning Guide

See [PROJECT_OVERVIEW.md](PROJECT_OVERVIEW.md) for a detailed breakdown of every task the playbook runs, what it configures, and the sysadmin reasoning behind each decision.

## Why Ansible?

Ansible lets you define your infrastructure as code — instead of manually SSHing into servers and running commands, you write repeatable, readable automation. This means your setup is documented, version-controlled, and reproducible.
