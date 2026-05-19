---
name: vps-initial-hardening
description: Use when setting up a new Linux VPS from scratch, migrating to SSH key-only authentication, hardening SSH config, installing fail2ban, or configuring UFW firewall on a fresh Ubuntu server.
---

# VPS Initial Hardening

## Overview

Step-by-step hardening workflow for a fresh Ubuntu 24.04 VPS. Covers SSH key auth, config hardening, UFW, fail2ban with custom preauth jail, unattended-upgrades, and optional Docker + panel deployment.

## Phase 1 — SSH Key Installation

**Do this before closing the initial password session.**

```bash
# On local machine — generate keypair
# Linux/macOS: run in terminal
# Windows: run in Git Bash, WSL, or PowerShell (ssh-keygen ships with Windows 10+)
ssh-keygen -t ed25519 -C "server-nickname" -f ~/.ssh/keyname
# Windows path equivalent: C:\Users\NAME\.ssh\keyname

# On VPS — install public key
mkdir -p /root/.ssh && chmod 700 /root/.ssh
echo "PASTE_PUBLIC_KEY_HERE" >> /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
```

Verify new key login works in a **second terminal** before proceeding.

## Phase 2 — SSH Config Hardening

**Ubuntu 24.04 cloud-init gotcha:** `/etc/ssh/sshd_config.d/50-cloud-init.conf` sets `PasswordAuthentication yes` and wins alphabetical sort. Disable it first.

```bash
# Disable cloud-init override
mv /etc/ssh/sshd_config.d/50-cloud-init.conf \
   /etc/ssh/sshd_config.d/50-cloud-init.conf.disabled

# Create hardening config
cat > /etc/ssh/sshd_config.d/99-harden.conf << 'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password
PermitEmptyPasswords no
EOF

# Reload (do NOT restart ssh.socket — kills active sessions)
systemctl reload ssh
```

**⚠️ CRITICAL — Ubuntu 24.04 ssh.socket:**
- `Port` directive in `sshd_config` does NOT work (socket activation overrides it)
- Never `systemctl restart ssh.socket` while connected — kills all sessions
- To change SSH port: disable `ssh.socket`, enable `ssh.service` standalone (needs QEMU console for recovery if it fails)

Verify password auth is rejected from a new terminal before closing existing session.

## Phase 3 — UFW Firewall

```bash
apt install -y ufw

# Allow needed ports BEFORE enabling
ufw allow 22/tcp comment "SSH"
# Add your application ports:
# ufw allow 443/tcp comment "VLESS+Reality"
# ufw allow 8443/udp comment "Hysteria2"
# ufw allow PANEL_PORT/tcp comment "Panel"

ufw default deny incoming
ufw default allow outgoing
ufw --force enable
ufw status verbose
```

## Phase 4 — fail2ban

```bash
apt install -y fail2ban

cat > /etc/fail2ban/jail.local << 'EOF'
[DEFAULT]
bantime  = 1h
findtime = 10m
maxretry = 5
backend  = systemd
# Add your own IPs to ignoreip:
# ignoreip = 127.0.0.1/8 ::1 YOUR_IP

[sshd]
enabled = true
port    = ssh
mode    = aggressive

[recidive]
enabled  = true
logpath  = /var/log/fail2ban.log
bantime  = 1w
findtime = 1d
maxretry = 3
EOF
```

### Custom `sshd-preauth-reset` Jail

Catches port-scanner "connection reset [preauth]" events that the default `sshd` jail misses entirely. Typically accounts for 30–50% of all blocks.

```bash
cat > /etc/fail2ban/filter.d/sshd-preauth-reset.conf << 'EOF'
[INCLUDES]
before = common.conf

[Definition]
_daemon = sshd
failregex = ^%(__prefix_line)sConnection (?:reset|closed) by (?:(?:authenticating|invalid) user \S+ )?<HOST> port \d+(?: \[preauth\])?\s*$
ignoreregex =
journalmatch = _SYSTEMD_UNIT=sshd.service + _COMM=sshd
EOF

cat > /etc/fail2ban/jail.d/sshd-preauth-reset.local << 'EOF'
[sshd-preauth-reset]
enabled          = true
filter           = sshd-preauth-reset
backend          = systemd
port             = ssh
findtime         = 10m
maxretry         = 3
bantime          = 4h
bantime.increment = true
bantime.maxtime  = 1w
EOF

systemctl enable --now fail2ban
```

Test filter works: `fail2ban-client status sshd-preauth-reset` should show jail active.

Debug: `fail2ban-regex systemd-journal /etc/fail2ban/filter.d/sshd-preauth-reset.conf --journalmatch "_SYSTEMD_UNIT=sshd.service + _COMM=sshd"` — expect non-zero matches on a live server.

## Phase 5 — Unattended Upgrades

```bash
apt install -y unattended-upgrades
dpkg-reconfigure -plow unattended-upgrades   # choose Yes

# Check it's enabled
systemctl is-active unattended-upgrades
```

Note: First-boot run may upgrade 100+ packages. Wait for lock release before further apt operations:
```bash
until ! fuser /var/lib/dpkg/lock-frontend >/dev/null 2>&1; do sleep 5; done
```

## Phase 6 — Docker + ZenithPanel (Optional)

```bash
# Install Docker
curl -fsSL https://get.docker.com | sh

# Deploy ZenithPanel with host networking (required — panel picks its own port)
docker run -d \
  --name zenithpanel \
  --restart unless-stopped \
  --network host \
  -v /opt/zenithpanel/data:/opt/zenithpanel/data \
  -v /var/run/docker.sock:/var/run/docker.sock \
  ghcr.io/harveyxiacn/zenithpanel:main

# Get assigned panel port
docker logs zenithpanel 2>&1 | grep -i "port\|listen\|start" | tail -5
# Then open that port in UFW
```

**Why `--network host`:** ZenithPanel auto-generates a random admin port stored in its SQLite DB. With `-p` port mapping, only pre-declared ports are exposed. `--network host` lets all auto-selected ports through.

## Phase 7 — Reboot

```bash
reboot   # Apply kernel/libc updates from unattended-upgrades
```

After reboot: verify SSH reconnects, fail2ban active, containers running, no reboot-pending flag.

## Checklist

- [ ] SSH key installed and tested
- [ ] `50-cloud-init.conf` disabled
- [ ] `99-harden.conf` active — password auth rejected
- [ ] UFW enabled, only needed ports open
- [ ] fail2ban + recidive + sshd-preauth-reset jails active
- [ ] unattended-upgrades enabled
- [ ] Rebooted to apply kernel update
- [ ] No `/var/run/reboot-required` flag after reboot

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Skip key verification before disabling passwords | Test key login in a second terminal first — always |
| `50-cloud-init.conf` still active | It wins alphabetical sort; rename it to `.disabled` |
| `systemctl restart ssh` (vs reload) | `restart` drops active connections; use `reload` |
| Restarting `ssh.socket` while connected | Kills all sessions on Ubuntu 24.04; use QEMU console for recovery |
| ZenithPanel with `-p PORT:PORT` mapping | Use `--network host`; panel port is auto-generated |
| apt lock during Docker install after unattended-upgrades | Wait for lock: `until ! fuser /var/lib/dpkg/lock-frontend ...` |
