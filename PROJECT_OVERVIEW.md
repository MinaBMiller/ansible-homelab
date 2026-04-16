# What This Playbook Does (and Why)

A breakdown of every task the playbook runs, what it configures on the server, and the sysadmin reasoning behind each decision. Each section ends with questions to test your understanding.

---

## Bootstrap Playbook (`playbooks/bootstrap.yml`)

Run once on a fresh server. After this, you never need a password again.

| What it does | Why |
|---|---|
| Creates an `ansible` user | Ansible needs its own account to SSH into the server. Using a dedicated user (instead of root or your personal account) keeps automation separate from human access — easier to audit and revoke. |
| Copies your SSH public key | SSH keys replace passwords. Your Mac holds the private key; the server holds the public key. If they match, you're in. Keys are longer and harder to brute-force than passwords. |
| Grants passwordless sudo | Ansible tasks often need root privileges (installing packages, editing system configs). Passwordless sudo lets Ansible escalate without prompting — necessary for unattended automation. The sudoers file is validated with `visudo` before writing to prevent lockouts. |

<details>
<summary><strong>Test yourself</strong></summary>

**Q: Why do we use `--ask-pass` only during bootstrap and never again?**
A: Bootstrap is the one time we don't have SSH key auth set up yet, so we need a password to get in. Once the playbook copies our public key to the server, all future connections use key-based auth — no password needed.

**Q: Why create a separate `ansible` user instead of just using your personal `mina` account?**
A: Separation of concerns. If automation runs under its own account, you can see exactly what Ansible changed (vs. what a human did) in logs and file ownership. You can also revoke Ansible's access without affecting your own login, or vice versa.

**Q: What would happen if `visudo` validation wasn't used when writing the sudoers file?**
A: A syntax error in a sudoers file can completely break `sudo` for the entire system. If that happens on a remote server with `PermitRootLogin no`, you've locked yourself out — the only recovery is booting into single-user/recovery mode. `visudo` checks syntax before the file is written and rejects bad configs.

**Q: The bootstrap playbook uses `lookup('file', '~/.ssh/id_ed25519.pub')`. What's the difference between `lookup('file', ...)` and `lookup('env', ...)`?**
A: `lookup('file', ...)` reads the contents of a file on the control machine (your Mac). `lookup('env', ...)` reads an environment variable on the control machine. The bootstrap reads the key file directly; the base playbook reads it from an env var (so the actual key isn't committed to git).

</details>

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

<details>
<summary><strong>Test yourself</strong></summary>

**Q: The firewall defaults to deny-incoming but allow-outgoing. Could a compromised server use outgoing connections maliciously? What would you do about it in a production environment?**
A: Yes — a compromised server could use outgoing connections to exfiltrate data, connect to a command-and-control server, or attack other systems. In production, you'd restrict outgoing traffic to only the ports and destinations needed (e.g., allow port 80/443 for package updates, port 53 for DNS, and block everything else). This is called egress filtering. In a homelab, the added complexity isn't worth it.

**Q: We set `PasswordAuthentication no` in SSH. What happens if you lose your SSH private key?**
A: You're locked out of remote access. Recovery options: boot into single-user/recovery mode from the VM console (UTM gives you console access), or re-enable password auth from the console. This is why you should always have a backup of your SSH key and/or physical/console access to the machine.

**Q: Why does the playbook set the timezone explicitly? Ubuntu already has a default timezone.**
A: Ubuntu defaults to UTC. While UTC is a valid choice (and common in production), the important thing is consistency — all servers should use the same timezone so you can correlate logs across machines. If server A says an error happened at 14:00 UTC and server B says 06:00 PST, you have to do mental math to know if they're related.

**Q: What's the difference between `apt upgrade` and `apt dist-upgrade`? Why does the playbook use `dist`?**
A: `upgrade` installs new versions but will never remove a package or install a new dependency. `dist-upgrade` (what Ansible calls `upgrade: dist`) is smarter — it can add or remove packages to resolve new dependencies. This matters for kernel updates and security patches that introduce new dependencies. `dist-upgrade` is safer for keeping a system fully patched.

**Q: Why is `fail2ban` useful if we've already disabled password authentication?**
A: Defense in depth. Even with password auth disabled, brute-force attempts still consume server resources (CPU, network, log space) and create noise in your logs. fail2ban reduces this by banning repeat offenders at the firewall level. It also protects other services that might use password auth in the future.

</details>

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

<details>
<summary><strong>Test yourself</strong></summary>

**Q: What does `exclusive: true` on the `authorized_key` module actually mean in practice? What's the risk of setting it to `false`?**
A: `exclusive: true` means the `authorized_keys` file will contain ONLY the keys Ansible manages — any manually-added keys are removed. With `false`, manually-added keys persist, which means a former employee or attacker who once had access could have added a key you don't know about. The tradeoff: with `exclusive: true`, you must manage ALL keys through Ansible — you can't quickly add a temporary key via SSH.

**Q: The `webrunner` account has `/sbin/nologin` as its shell. If you needed to debug something running as `webrunner`, how would you run commands as that user?**
A: Use `sudo -u webrunner <command>` from an account with sudo access. For example: `sudo -u webrunner whoami`. You can also use `sudo -u webrunner -s /bin/bash` to get a shell as that user, bypassing the nologin restriction because sudo overrides the shell setting.

**Q: Why use `/etc/sudoers.d/` drop-in files instead of editing the main `/etc/sudoers` file directly?**
A: Three reasons: (1) Each user's sudo config is in its own file, so revoking access means deleting one file instead of editing a shared file. (2) Multiple tools/roles can manage sudo without conflicting — they each write their own file. (3) If a drop-in file has a syntax error, it only breaks that file, not the entire sudoers configuration.

**Q: The permission `0700` on home directories means only the owner can access them. What permission would you use if you wanted the owner's group to also read files but not modify them?**
A: `0750`. The `5` for group means read (4) + execute (1) — execute on a directory means you can `cd` into it and list files. Without the execute bit, even with read permission, you can't access the directory contents.

**Q: Why must groups be created before users in the task ordering?**
A: The user creation task assigns users to groups with `groups: [ops, sudo]`. If those groups don't exist yet, the task fails. Ansible runs tasks top-to-bottom, so the group creation task must come first. This is one of the few cases where order matters in declarative configuration.

</details>

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

<details>
<summary><strong>Test yourself</strong></summary>

**Q: We experienced the VM IP changing between reboots during setup. How would enabling a static IP have prevented that? What's the tradeoff?**
A: With DHCP, the router assigns whatever IP is available — it can change each time. A static IP is hardcoded in the VM's network config, so it never changes. The tradeoff: you must pick an IP outside the DHCP range (or reserve it on the router) to avoid conflicts. If two devices claim the same IP, neither works correctly.

**Q: Why does `/etc/hosts` get checked before DNS? When would you want to override DNS with a hosts entry?**
A: The lookup order is defined in `/etc/nsswitch.conf`. Checking hosts first is useful for: (1) local name resolution without a DNS server (like in a homelab), (2) overriding public DNS for testing (pointing `myapp.com` at a local dev server), (3) blocking domains by pointing them to `127.0.0.1`. It's faster too — no network request needed.

**Q: The role uses two DNS providers (Cloudflare and Google). Why not just use one?**
A: Redundancy. If Cloudflare's DNS goes down, the server falls back to Google. DNS is critical infrastructure — without it, almost nothing works (package installs, git, web requests all fail). Using two independent providers from different companies protects against a single provider's outage.

**Q: The Netplan config file uses mode `0600`. Why is it more restrictive than most other config files (which use `0644`)?**
A: Netplan files can contain sensitive network details like static IPs, gateways, and potentially WiFi passwords or VLAN configurations. `0600` means only root can read it. A regular user doesn't need to see network configuration, and exposing it could help an attacker map your network.

**Q: What would happen if you set `configure_static_ip: true` but used an IP that's already taken by another device on the network?**
A: You'd get an IP conflict. Both devices would intermittently lose connectivity — packets would sometimes go to one, sometimes the other. Network tools like `arping` can detect this. The fix is either to choose a different IP or remove the conflicting device.

</details>

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

<details>
<summary><strong>Test yourself</strong></summary>

**Q: We hit this exact issue during setup — SSH was `active (dead)` and wouldn't start on boot. What's the difference between `systemctl start` and `systemctl enable`, and why do you need both?**
A: `start` runs the service right now. `enable` creates a symlink so systemd starts it automatically on boot. Without `enable`, the service stops on reboot and you have to manually start it again — exactly what happened with SSH during our setup. You almost always want both.

**Q: Why does systemd require `daemon-reload` after changing a unit file? Why doesn't it just watch for file changes?**
A: Systemd reads and caches all unit files at boot for performance. Watching thousands of files for changes would add overhead and introduce race conditions (what if you're halfway through writing a file?). `daemon-reload` gives you explicit control — you make your changes, then tell systemd "I'm done, re-read everything." It's a deliberate design choice favoring reliability over convenience.

**Q: The journal is set to keep 1 month of logs with a 500MB cap. If the server generates 600MB of logs in two weeks, what happens?**
A: The size limit (500MB) takes precedence. Journald will delete the oldest logs to stay under 500MB, even if they're less than a month old. Size limits protect against disk-full scenarios; time limits are secondary cleanup. This is why both exist — the size limit is the safety net, the time limit is housekeeping.

**Q: Why disable `snapd` specifically? Are there other default Ubuntu services you might consider disabling on a server?**
A: Snapd runs background processes (snapd, snap updates) that consume memory and CPU on a server that doesn't use snap packages. Other candidates for disabling: `cloud-init` (if not in a cloud environment), `ModemManager` (manages cellular modems — useless on a server), `cups` (printing — not needed on headless servers). The principle: every running service is attack surface and resource consumption. If you don't need it, turn it off.

**Q: A service is `enabled` but shows `inactive (dead)`. What are some possible reasons?**
A: Several possibilities: (1) It crashed after starting — check `journalctl -u <service>` for error logs. (2) Its dependencies aren't met — another service it depends on isn't running. (3) It was stopped manually with `systemctl stop`. (4) Its `ExecStart` command doesn't exist or isn't executable. (5) It's a "oneshot" type that runs and exits (this is normal for setup tasks). `systemctl status` and journal logs are your first debugging tools.

</details>

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

<details>
<summary><strong>Test yourself</strong></summary>

**Q: Node Exporter exposes metrics on port 9100 with no authentication. Why is this acceptable in a homelab but dangerous in production?**
A: In a homelab on an isolated network, only you can reach port 9100. In production, unauthenticated metrics expose detailed system information (CPU count, disk layout, running services, memory) that helps attackers plan their approach. Production setups use either firewall rules restricting access to the Prometheus server's IP, a reverse proxy with authentication, or TLS client certificates.

**Q: Why run Node Exporter as a dedicated `node_exporter` user with `nologin` instead of just running it as root?**
A: Principle of least privilege. Node Exporter only needs to read `/proc` and `/sys` (which are world-readable). If it ran as root and had a vulnerability, an attacker gets full system access. As `node_exporter` with `nologin`, a compromised process can only read system stats — it can't modify files, install software, or access other users' data.

**Q: The role downloads Node Exporter as a binary from GitHub instead of installing it via `apt`. What are the advantages and disadvantages of each approach?**
A: Binary from GitHub: you get the exact version you want, works on any Linux distro, and updates are controlled by changing a version variable. But you're responsible for updating it yourself — no automatic security patches. Via apt: updates come through the normal package manager, but you're limited to whatever version Ubuntu packages (often outdated), and it may not be available in the default repos at all.

**Q: What's the difference between Node Exporter (metrics) and Logwatch (log digest)? When would you use each?**
A: Node Exporter gives you real-time numeric data — CPU at 80%, disk 90% full, 1000 network packets/sec. It's for dashboards, alerting, and trend analysis. Logwatch gives you a human-readable summary of what happened in the logs — 47 SSH login attempts, 3 sudo commands, 2 package installs. Use metrics for "is something wrong right now?" and logs for "what happened yesterday?" They complement each other.

**Q: You can see Node Exporter metrics by visiting `http://<server-ip>:9100/metrics` in a browser. What would you need to add to actually graph and alert on those metrics?**
A: Prometheus — a time-series database that scrapes the `/metrics` endpoint on a schedule (e.g., every 15 seconds), stores the data, and lets you query it. For visualization, add Grafana — it connects to Prometheus and lets you build dashboards. For alerting, configure Prometheus Alertmanager with rules like "alert if CPU > 90% for 5 minutes." This is the standard Prometheus + Grafana + Alertmanager stack.

</details>

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

<details>
<summary><strong>Test yourself</strong></summary>

**Q: We hit a bug where `X-XSS-Protection 1; mode=block` broke nginx. What was the actual problem, and what does this teach you about configuration templating?**
A: The Jinja2 template rendered `add_header X-XSS-Protection 1; mode=block;` — nginx parsed the semicolon as a statement terminator, so it saw `mode=block` as a separate (invalid) directive. The fix was to quote the header value: `add_header X-XSS-Protection "1; mode=block";`. Lesson: when templating config files, always consider how special characters in your data interact with the config file's syntax. Quoting values is defensive — it's always safer to quote than to assume values won't contain special characters.

**Q: What's the difference between a web server and a reverse proxy? Why might you use nginx as a reverse proxy instead of exposing your app directly?**
A: A web server serves files from disk (HTML, CSS, images). A reverse proxy sits between clients and your application, forwarding requests. Reasons to use a reverse proxy: (1) SSL/TLS termination — nginx handles HTTPS so your app doesn't have to. (2) Static file serving — nginx is much faster at serving static files than most app servers. (3) Load balancing — distribute traffic across multiple app instances. (4) Security — nginx can filter malicious requests before they reach your app. (5) Your app can listen on localhost only, reducing attack surface.

**Q: The role uses `validate: nginx -t -c %s` when deploying the config. What does this do, and what would happen without it?**
A: `nginx -t` tests the config file for syntax errors without applying it. Ansible runs this check before writing the file — if validation fails, the old config stays in place and nginx keeps running. Without it, a bad config would be written, and when nginx tries to reload, it would crash. The `%s` is a placeholder that Ansible replaces with the path to the temporary file being validated.

**Q: Why does the role open port 443 (HTTPS) even though SSL isn't configured yet?**
A: Forward-thinking configuration. When you add SSL later, the firewall is already ready — you just need to configure the certificate in nginx. If port 443 wasn't open, adding SSL would require changes to both the nginx role AND the firewall, which is easy to forget. Opening an unused port has minimal risk — there's nothing listening on it, so connections are simply refused.

**Q: The `sites-available` / `sites-enabled` pattern uses symlinks. Why is this better than just putting configs directly in one directory?**
A: It separates "exists" from "active." You can disable a site by removing the symlink and re-enable it by recreating it — no copying, no risk of losing the config. This is useful for maintenance (temporarily disable a site), staging (have a config ready but not active), and rollback (keep the old config in sites-available while the new one is active).

</details>

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

---

## Big Picture Questions

<details>
<summary><strong>Think about these</strong></summary>

**Q: If you deleted the VM and created a fresh one, what steps would you need to reproduce the entire setup?**
A: (1) Install Ubuntu Server, create a user. (2) Copy `hosts.ini.example` to `hosts.ini` with the new VM's IP. (3) Set up `.env` with your SSH public key. (4) Run bootstrap with `--ask-pass --ask-become-pass`. (5) Run `base.yml`. That's it — two commands and your server is fully configured. This is the core value of infrastructure as code: your setup is documented, version-controlled, and reproducible.

**Q: The playbook currently targets `hosts: all`. What would you change if you had three servers and only wanted nginx on one of them?**
A: Create host groups in your inventory (e.g., `[webservers]`, `[databases]`). Then either split `base.yml` into multiple plays targeting different groups, or use `when` conditions based on group membership. You could also create separate playbooks like `web.yml` that only includes the nginx role and targets `[webservers]`.

**Q: What's the risk of running `ansible-playbook base.yml` on a server that's already configured? Will it break anything?**
A: Ansible is idempotent — running the same playbook twice produces the same result. Tasks check the current state before acting. If a package is already installed, `apt` skips it. If a file already has the right content, it's not rewritten. Handlers only fire when a task reports a change. You should be able to run the playbook daily without fear. This is a fundamental Ansible design principle.

**Q: You now have firewall, SSH hardening, fail2ban, and security headers. What's still missing from a security perspective?**
A: Some things to consider: (1) SSL/TLS certificates for nginx (currently HTTP only). (2) Log aggregation to a central server (logs on a compromised server can be tampered with). (3) Intrusion detection (like AIDE or OSSEC). (4) Regular backups and tested restore procedures. (5) Network segmentation (separate VLANs for different services). (6) Secrets management (Ansible Vault for sensitive variables). (7) Regular vulnerability scanning. Security is layers — what we have is a solid foundation, not a complete solution.

**Q: How would you test that the playbook actually works correctly, beyond just seeing `ok=45 changed=9`?**
A: Write verification tasks or a separate test playbook that checks: (1) `curl http://localhost` returns the nginx page. (2) `ufw status` shows the expected rules. (3) SSH with password fails (password auth disabled). (4) `curl http://localhost:9100/metrics` returns Node Exporter data. (5) Each expected user exists and has the right groups. (6) Expected services are running. This is infrastructure testing — tools like Molecule, Testinfra, or ServerSpec automate this.

</details>
