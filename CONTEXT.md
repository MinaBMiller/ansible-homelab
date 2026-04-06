# Project Context — for Claude

## What this is
An Ansible homelab project built to develop sysadmin and networking skills.
The Mac is both the control node (where Ansible runs) and the hypervisor (UTM runs VMs).

## User setup
- Apple Silicon Mac
- UTM installed, Ubuntu Server 24.04 LTS ARM64 VM created and installed
- VM IP: 192.168.64.255
- VM has an account: `mina` (password set during Ubuntu install)
- SSH key generated on Mac at ~/.ssh/id_ed25519
- SSH public key added to inventory/group_vars/all.yml under the mina user

## What's been built
Six Ansible roles, all wired into playbooks/base.yml:

| Role       | What it does                                                  |
|------------|---------------------------------------------------------------|
| common     | apt updates, base packages, UFW firewall, SSH hardening, timezone (PST) |
| users      | user accounts, groups, SSH keys, sudo, home dir permissions   |
| networking | hostname, DNS (1.1.1.1/8.8.8.8), /etc/hosts, static IP via Netplan |
| systemd    | journald log limits, service management, custom unit template |
| monitoring | Node Exporter (port 9100) as systemd service, logwatch        |
| nginx      | web server, vhost template, reverse proxy support, TLS-ready  |

## Where we are right now
Trying to run the bootstrap playbook. Hit two issues already fixed:
1. community.general collection was missing → added requirements.yml
2. stdout_callback = yaml was deprecated → changed to ansible.builtin.yaml in ansible.cfg

## What needs to happen next (on the Mac)
1. Pull latest: `git pull`
2. Install collection: `ansible-galaxy collection install -r requirements.yml`
3. Run bootstrap (one-time, uses password): `ansible-playbook playbooks/bootstrap.yml --ask-pass -u mina`
4. Run base playbook: `ansible-playbook playbooks/base.yml`

## Key files to know
- `inventory/hosts.ini` — set ubuntu-01 IP to 192.168.64.255 (not committed to git)
- `inventory/group_vars/all.yml` — mina's SSH key goes here
- `playbooks/bootstrap.yml` — run once as the mina user with --ask-pass
- `playbooks/base.yml` — main playbook, runs all roles

## Conventions used
- Roles follow: defaults/ (variables) → tasks/ (work) → handlers/ (triggered restarts) → templates/ (Jinja2)
- Variables defined in defaults/main.yml, overridden in group_vars/all.yml
- Handlers only fire when a task reports a change
- All playbook comments explain the "why", not just the "what"
