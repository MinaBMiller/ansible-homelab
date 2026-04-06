# Architecture

## Infrastructure Overview

```
┌─────────────────────────────────────────────────────────────┐
│                        Mac (Control Node)                   │
│                                                             │
│   ansible-homelab/          You write playbooks here.       │
│   Ansible installed         Ansible SSHes into VMs and      │
│   SSH key pair              runs tasks remotely.            │
└──────────────────────────────────┬──────────────────────────┘
                                   │ SSH
                    ┌──────────────▼──────────────┐
                    │         UTM Hypervisor       │
                    │                             │
                    │  ┌─────────────────────┐    │
                    │  │     ubuntu-01        │    │
                    │  │   192.168.64.10      │    │
                    │  │                     │    │
                    │  │  - ansible user      │    │
                    │  │  - UFW firewall      │    │
                    │  │  - SSH hardened      │    │
                    │  └─────────────────────┘    │
                    │                             │
                    │  ┌─────────────────────┐    │
                    │  │     ubuntu-02        │    │
                    │  │   192.168.64.11      │    │
                    │  │   (future)           │    │
                    │  └─────────────────────┘    │
                    └─────────────────────────────┘
```

## Playbook Execution Flow

```
ansible-playbook playbooks/bootstrap.yml     (run once, as root)
        │
        ▼
┌───────────────────┐
│    bootstrap      │  Creates 'ansible' user
│                   │  Copies SSH public key
│                   │  Grants passwordless sudo
└───────────────────┘

ansible-playbook playbooks/base.yml          (run any time after bootstrap)
        │
        ├──▶ role: common
        │         │
        │         ├── Update & upgrade packages
        │         ├── Install base tools (curl, vim, git, fail2ban, ufw)
        │         ├── Set timezone (America/Los_Angeles)
        │         ├── Configure UFW firewall
        │         │     deny all incoming
        │         │     allow SSH (port 22)
        │         ├── Harden SSH
        │         │     no root login
        │         │     no password auth (keys only)
        │         └── Enable automatic security updates
        │
        ├──▶ role: users
        │         │
        │         ├── Create groups (ops, developers)
        │         ├── Create user accounts
        │         ├── Add SSH authorized keys
        │         ├── Grant/revoke sudo per user
        │         └── Lock down home directories (0700)
        │
        ├──▶ role: networking
        │         │
        │         ├── Set hostname
        │         ├── Configure DNS resolvers (1.1.1.1, 8.8.8.8)
        │         ├── Write /etc/hosts entries
        │         └── Configure static IP via Netplan (if enabled)
        │
        └──▶ role: systemd
                  │
                  ├── Configure journald log limits and retention
                  ├── Manage built-in services (fail2ban, ufw, snapd)
                  └── Deploy and enable custom service units
```

## Project File Structure

```
ansible-homelab/
│
├── ansible.cfg                  Global settings (inventory path, SSH user)
│
├── inventory/
│   ├── hosts.ini                Defines servers and groups
│   └── group_vars/
│       └── all.yml              Variables applied to every server
│                                (users, SSH keys, etc.)
│
├── playbooks/
│   ├── bootstrap.yml            First-run setup (run once as root)
│   └── base.yml                 Main playbook — applies all roles
│
└── roles/                       Self-contained bundles of tasks
    │
    ├── common/
    │   ├── defaults/main.yml    Default variables (packages, timezone, SSH port)
    │   ├── tasks/main.yml       System updates, firewall, SSH hardening
    │   └── handlers/main.yml    Restart SSH (only when config changes)
    │
    ├── users/
    │   ├── defaults/main.yml    Default user/group variable shapes
    │   └── tasks/main.yml       Groups → users → SSH keys → sudo → permissions
    │
    └── networking/
        ├── defaults/main.yml    Hostname, DNS servers, static IP settings
        ├── tasks/main.yml       Hostname, DNS, /etc/hosts, Netplan
        ├── handlers/main.yml    Restart resolved, apply Netplan
        └── templates/
            └── netplan.j2       Jinja2 template for static IP config
```

## Roles Planned

| Role         | Status      | Purpose                                      |
|--------------|-------------|----------------------------------------------|
| common       | ✅ Built    | System updates, firewall, SSH hardening       |
| users        | ✅ Built    | User accounts, groups, SSH keys, sudo         |
| networking   | ✅ Built    | Hostname, DNS, static IP, /etc/hosts          |
| systemd      | ✅ Built    | Service management, custom units, journald    |
| monitoring   | 🔜 Planned  | Metrics, log aggregation, alerting            |
| nginx        | 🔜 Planned  | Web server, reverse proxy, TLS               |
