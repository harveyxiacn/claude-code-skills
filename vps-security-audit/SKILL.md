---
name: vps-security-audit
description: Use when checking security status of one or more Linux VPS servers, running periodic health checks, auditing fail2ban effectiveness, investigating SSH attacks, or verifying hardening measures are working after setup.
---

# VPS Security Audit

## Overview

Structured workflow for auditing Linux VPS security. Covers SSH tunnel access, fail2ban health, system state, and service checks — produces a formatted security report.

## Audit Commands (run per server)

```bash
# Identity + uptime
hostname && uptime

# Reboot pending?
test -f /var/run/reboot-required \
  && echo "⚠️ REBOOT PENDING: $(cat /var/run/reboot-required)" \
  || echo "no reboot pending"
uname -r

# fail2ban jail overview
fail2ban-client status
fail2ban-client status sshd | grep -E 'Currently|Total'
fail2ban-client status sshd-preauth-reset 2>/dev/null | grep -E 'Currently|Total'
fail2ban-client status recidive 2>/dev/null | grep -E 'Currently|Total'

# Bans in last 24h
grep "Ban " /var/log/fail2ban.log \
  | awk -v d="$(date -d '24 hours ago' '+%Y-%m-%d %H:%M')" '$0 >= d' | tail -30

# 24h SSH probe count
journalctl -u sshd --since "24 hours ago" \
  | grep -cE "Failed|Invalid|preauth|Connection reset" || echo 0

# Disk + memory
df -h / && free -h

# Key services
systemctl is-active docker fail2ban ufw 2>/dev/null
ufw status verbose 2>/dev/null | head -25

# Docker containers
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Image}}" 2>/dev/null
```

## SSH Tunnel Access Pattern

When a local TUN proxy (sing-box, clash, etc.) intercepts direct VPS connections, forward a local port to the VPS first, then SSH through it:

**Windows (Xshell):** Add a local port forwarding rule: `127.0.0.1:LOCAL_PORT → VPS_IP:22`

**Linux / macOS:** Open tunnel in background:
```bash
ssh -i ~/.ssh/KEY -fNL LOCAL_PORT:VPS_IP:22 jump-host-or-direct
```

**Connect (all platforms):**
```bash
ssh -i ~/.ssh/KEY -p LOCAL_PORT root@127.0.0.1
```

Key path: `~/.ssh/KEY` on Linux/macOS; `C:\Users\NAME\.ssh\KEY` on Windows (Git Bash: `~/.ssh/KEY` also works).

List available keys: `ls ~/.ssh/`

## Report Format

```
## Security Audit — <SERVER> (<DATE>)

| Item           | Status                              |
|----------------|-------------------------------------|
| Uptime         | X days                              |
| Reboot pending | ✅ No / ⚠️ Yes (kernel update)       |
| Kernel         | X.X.X-XX-generic                   |
| Disk /         | XX% used (XG / XG)                  |
| Memory         | XXX used / XG total                 |

fail2ban (total bans / currently banned):
- sshd:               X total / X active
- sshd-preauth-reset: X total / X active
- recidive:           X total / X active

Services: docker ✅  fail2ban ✅  ufw ✅

⚠️ Alerts: [disk >80%, service down, suspicious IPs, etc.]
```

## Multi-Server Parallel Check

Dispatch one Bash call per server simultaneously — each with its own key and tunnel port:

```bash
# Server A (tunnel on 2222)
ssh -i ~/.ssh/KEY_A -p 2222 root@127.0.0.1 \
  "hostname; uptime; fail2ban-client status sshd | grep Total"

# Server B (tunnel on 2223) — run in parallel
ssh -i ~/.ssh/KEY_B -p 2223 root@127.0.0.1 \
  "hostname; uptime; fail2ban-client status sshd | grep Total"
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Direct SSH when TUN proxy is active | Use Xshell local port forward |
| `journalmatch = _SYSTEMD_UNIT=ssh.service` | Must be `sshd.service` on Ubuntu 24.04 |
| fail2ban-regex returns 0 matches | Add `^%(__prefix_line)s` to failregex start |
| x-ui/3x-ui not checked | `systemctl is-active x-ui` — backup VPN may silently die |
| Disk ignored | Alert if `/` >75%; Docker image cache can fill disk quietly |
