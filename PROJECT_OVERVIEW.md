# What This Playbook Does (and Why)

A breakdown of every task the playbook runs, what it configures on the server, and the sysadmin reasoning behind each decision.

---

## Bootstrap Playbook (`playbooks/bootstrap.yml`)

Run once on a fresh server. After this, you never need a password again.

| What it does | Why |
|---|---|
| Creates an `ansible` user | Ansible needs its own account to SSH into the server. Using a dedicated user (instead of root or your personal account) keeps automation separate from human access — easier to audit and revoke. |
| Copies your SSH public key | SSH keys replace passwords. Your Mac holds the private key; the server holds the public key. If they match, you're in. Keys are longer and harder to brute-force than passwords. |
| Grants passwordless sudo | Ansible tasks often need root privileges (installing packages, editing system configs). Passwordless sudo lets Ansible escalate without prompting — necessary for unattended automation. The sudoers file is validated with `visudo` before writing to prevent lockouts. |

---

## Base Playbook (`playbooks/base.yml`)

The main playbook. Applies all six roles to every server in your inventory.

---

### 1. Common Role — System Baseline

The foundation every server needs before anything else.

#### System Updates

| What | Why |
|---|---|
| Updates the apt package cache | apt maintains a local index of available packages. If the cache is stale, you might install outdated or missing versions. `cache_valid_time: 3600` avoids redundant refreshes within an hour. |
| Upgrades all packages (`dist` upgrade) | Keeps the system patched. `dist` (as opposed to `full`) handles dependency changes safely — it can add or remove packages to satisfy new dependencies, which is important for kernel and security updates. |

#### Base Packages

| Package | What it does |
|---|---|
| `curl` | Command-line tool for making HTTP requests. Essential for downloading files, testing APIs, and debugging network issues. |
| `vim` | Text editor. When you SSH into a server, you need to be able to edit config files. |
| `git` | Version control. Useful for pulling code, configs, or scripts onto the server. |
| `ufw` | Uncomplicated Firewall — a user-friendly frontend for iptables (the Linux kernel's packet filtering system). |
| `fail2ban` | Monitors log files for repeated failed login attempts and temporarily bans the offending IP. Stops brute-force SSH attacks. |
| `unattended-upgrades` | Automatically installs security patches. Servers you don't log into daily still get critical fixes. |

#### Timezone

Sets the system clock to `America/Los_Angeles` (PST/PDT). Consistent timezones across servers make log correlation possible — if server A logs an event at 14:00 and server B at 14:00, you know they happened at the same time.

#### Firewall (UFW)

| Rule | Why |
|---|---|
| Default deny incoming | The safest starting point: nothing gets in unless you explicitly allow it. This is the "allowlist" approach — the opposite of trying to block known-bad traffic. |
| Default allow outgoing | The server needs to reach the internet for package updates, DNS, NTP, etc. Restricting outgoing traffic is possible but adds complexity that isn't worth it for a homelab. |
| Allow SSH (port 22) | Without this rule, the firewall would lock you out of your own server. SSH is the only way to manage a headless (no monitor) server remotely. |

#### SSH Hardening

| Setting | Why |
|---|---|
| `PermitRootLogin no` | The root account exists on every Linux system and has unlimited power. Disabling root login forces attackers to guess both a username AND credentials, and ensures all access goes through named accounts (better audit trail). |
| `PasswordAuthentication no` | Passwords can be brute-forced. SSH keys can't (practically). Once key auth is set up, disabling passwords eliminates an entire class of attacks. |
| `X11Forwarding no` | X11 forwarding lets you run graphical apps over SSH. On a headless server there are no graphical apps — leaving it on just adds attack surface. |

#### Automatic Security Updates

Writes an apt config that checks for and installs security updates daily. Unattended upgrades are a safety net — they patch known vulnerabilities even if you forget to manually update.

---

### 2. Users Role — Accounts and Access Control

#### Groups

Creates groups like `ops` and `developers` before creating users (groups must exist first). Each group gets an explicit GID (Group ID) so the number stays consistent across servers — important when sharing files over a network (NFS uses numeric IDs, not names).

#### User Accounts

| Concept | Explanation |
|---|---|
| UIDs | Every Linux user has a numeric User ID. UIDs under 1000 are reserved for system accounts. Human/service accounts you create use 1000+. Pinning UIDs ensures consistency across multiple servers. |
| `append: true` | Adds the user to the listed groups without removing them from any existing groups. Without `append`, Ansible would replace all group memberships. |
| `/sbin/nologin` shell | Used for service accounts (like `webrunner`). Even if someone steals the credentials, they can't get an interactive shell. The account exists only to own processes and files. |
| `state: present/absent` | Ansible is declarative — you describe the desired state, not the steps. `present` means "this user should exist" and `absent` means "remove this user." |

#### SSH Keys

Deploys public keys to `~/.ssh/authorized_keys` for each user. The `exclusive: true` flag removes any keys not in the Ansible-managed list — if someone manually adds a key to the server, Ansible will remove it on the next run. This prevents stale access.

#### Sudo Access

Instead of adding users to the `sudo` group, creates individual files under `/etc/sudoers.d/`. One file per user means you can grant or revoke sudo access for a single person without touching the main sudoers file. Every file is validated with `visudo` before writing — a syntax error in sudoers can lock everyone out of sudo.

#### Home Directory Permissions

Sets home directories to `0700` (owner-only access). The three digits represent owner/group/other permissions. `7` = read+write+execute, `0` = no access. There's no reason for one user to browse another's home directory.

---

### 3. Networking Role — Identity and Connectivity

#### Hostname

Sets the server's name (e.g., `ubuntu-01`). The hostname appears in:
- Your shell prompt (`mina@ubuntu-01:~$`)
- System logs (so you can tell which server generated a log entry)
- Monitoring dashboards

Written to both the running kernel (`hostname` module) and `/etc/hostname` (persists across reboots).

#### /etc/hosts

A local name-to-IP mapping file checked *before* DNS. Adding your homelab servers here lets them find each other by name without needing a DNS server. The `blockinfile` module writes a clearly marked section so Ansible's entries don't clash with the system defaults.

**Lookup order:** `/etc/hosts` → DNS → other sources (defined in `/etc/nsswitch.conf`).

#### DNS Resolvers

Configures `systemd-resolved` to use:
- **1.1.1.1** (Cloudflare) — fast, privacy-focused
- **8.8.8.8** (Google) — reliable fallback

DNS resolvers translate domain names (like `google.com`) into IP addresses. Without working DNS, the server can't install packages, pull from git, or reach anything by name.

#### Static IP (Optional)

By default, the VM gets a dynamic IP from DHCP — the IP can change on every reboot. A static IP means the address in your inventory file always works. Configured via Netplan (Ubuntu's network configuration tool). Disabled by default (`configure_static_ip: false`).

---

### 4. Systemd Role — Service Management

#### What is systemd?

Systemd is the init system on modern Linux. It's the first process that runs (PID 1) and manages every other service. Key concepts:

| Concept | Explanation |
|---|---|
| **Unit** | A thing systemd manages — usually a service (`.service`), but also timers, mounts, etc. |
| **enabled** | The service starts automatically on boot. Survives reboot. |
| **started** | The service is running right now. Doesn't survive reboot on its own. |
| **daemon-reload** | Systemd caches unit definitions in memory. After adding or changing a `.service` file, you must run `daemon-reload` so systemd reads the new version. |

#### Journal Configuration

Journald stores system logs in binary format under `/var/log/journal`. Without limits, logs can fill your disk. The role caps:
- Total log size: 500MB
- Single file size: 50MB (rotated after)
- Retention: 1 month
- Free space reserve: 1GB minimum

#### Managed Services

| Service | State | Why |
|---|---|---|
| `fail2ban` | started + enabled | Brute-force protection should always be running. |
| `ufw` | started + enabled | Firewall must survive reboots. |
| `unattended-upgrades` | started + enabled | Auto-security-patches need to be on at all times. |
| `snapd` | stopped + disabled | Snap is Ubuntu's app packaging system. Not needed in a minimal server setup — it consumes resources and adds attack surface. |

#### Custom Service Units

A Jinja2 template for creating your own systemd services. Empty by default — when you have a script or app you want to run as a managed service, you define it in variables and Ansible generates the `.service` file, reloads systemd, and starts it.

---

### 5. Monitoring Role — Visibility into Your Server

#### Node Exporter

| Concept | Explanation |
|---|---|
| What it is | A lightweight agent from the Prometheus project. Runs as an HTTP server on port 9100. |
| What it does | Exposes system metrics (CPU, RAM, disk, network) at `http://<server-ip>:9100/metrics` in a format Prometheus can scrape. |
| Why a dedicated user | `node_exporter` runs as its own system user with `/sbin/nologin`. Principle of least privilege — if the process is compromised, the attacker only has the permissions of that locked-down account. |
| Why a static binary | Node Exporter is written in Go and compiles to a single binary with zero dependencies. No package manager, no runtime, no library conflicts. Download and run. |
| ARM64 build | UTM on Apple Silicon runs ARM Linux VMs. On Intel servers you'd use the `amd64` build. |

**Collectors enabled:** CPU usage, disk I/O, disk space, load averages, memory, network traffic, systemd unit states, and clock drift.

#### Logwatch

Parses system logs daily and produces a human-readable summary. Runs via cron and sends output to local root mail (`/var/mail/root`). It's a lightweight alternative to a full log aggregation stack — gives you a daily "what happened on this server" digest.

---

### 6. Nginx Role — Web Server and Reverse Proxy

#### What is Nginx?

A high-performance web server that can also act as a reverse proxy (sit in front of an application and forward requests to it, handling SSL/TLS and load balancing).

#### Configuration

| Setting | Why |
|---|---|
| `sendfile on` | Uses kernel-level file transfer instead of copying through userspace. Much faster for static files. |
| `tcp_nopush` + `tcp_nodelay` | Batch packet headers, then send data immediately. Optimizes both throughput and latency. |
| `keepalive_timeout 65` | Keeps connections open for 65 seconds waiting for another request. Avoids the overhead of re-establishing TCP connections for the same client. |
| `server_tokens off` | Hides the nginx version number from error pages and headers. Knowing the version helps attackers look up known vulnerabilities. |
| `gzip on` | Compresses responses before sending. Reduces bandwidth and speeds up page loads. The browser decompresses transparently. |

#### Security Headers

| Header | Protection |
|---|---|
| `X-Frame-Options: SAMEORIGIN` | Prevents clickjacking — other sites can't embed your pages in an iframe. |
| `X-Content-Type-Options: nosniff` | Prevents MIME sniffing — browsers must respect the declared content type. Stops attacks where a malicious file is disguised as a different type. |
| `X-XSS-Protection: 1; mode=block` | Activates the browser's built-in XSS filter. Legacy but still provides a layer of defense. |
| `Referrer-Policy: no-referrer-when-downgrade` | Controls what URL info is sent in the Referer header. Stops leaking internal URLs when navigating to external sites over HTTP. |

#### Virtual Hosts (vhosts)

Nginx uses a two-directory convention:
- `sites-available/` — all site configs (active or not)
- `sites-enabled/` — symlinks to configs that are actually active

Enabling a site = creating a symlink. Disabling = removing it. This lets you toggle sites without deleting their configuration.

The role creates a default vhost serving a placeholder page on port 80, with the web root at `/var/www/html`.

#### Firewall Rules

Opens ports 80 (HTTP) and 443 (HTTPS) in UFW. The common role set the default to deny-all — each service role is responsible for opening only the ports it needs.

---

## How the Pieces Connect

```
Mac (control node)
  └── ansible-playbook base.yml
        └── SSH → ubuntu-01 (target VM)
              ├── common    → patches, firewall, SSH lockdown
              ├── users     → accounts, keys, sudo
              ├── networking → hostname, DNS, /etc/hosts
              ├── systemd   → log limits, service states
              ├── monitoring → Node Exporter :9100, logwatch
              └── nginx     → web server :80, reverse proxy ready
```

The roles run in order and build on each other:
1. **Common** sets up the firewall with deny-all
2. **Users** creates accounts that can manage the system
3. **Networking** gives the server a stable identity
4. **Systemd** ensures critical services are running and logs are managed
5. **Monitoring** adds observability (you can't fix what you can't see)
6. **Nginx** opens specific ports through the firewall common established

Each role follows the same structure: `defaults/` (variables) → `tasks/` (work) → `handlers/` (triggered restarts) → `templates/` (Jinja2 config files). Variables in `defaults/` are the role's suggestions; override them in `group_vars/all.yml` to customize for your environment.
