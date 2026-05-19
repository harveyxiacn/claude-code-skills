# claude-code-skills

A collection of Claude Code skills for Linux VPS management — SSH hardening, fail2ban configuration, and periodic security auditing.

## Skills

### [`vps-initial-hardening`](./vps-initial-hardening/SKILL.md)

Step-by-step hardening workflow for a fresh **Ubuntu 24.04** VPS:

- SSH key-only authentication (disable password auth, fix cloud-init override)
- UFW firewall setup
- fail2ban with `sshd`, `recidive`, and custom `sshd-preauth-reset` jails
- Unattended upgrades
- Optional: Docker + ZenithPanel deployment

Includes critical Ubuntu 24.04 gotchas (ssh.socket activation, 50-cloud-init.conf alphabetical sort issue).

### [`vps-security-audit`](./vps-security-audit/SKILL.md)

Periodic security check workflow for one or more VPS servers:

- fail2ban jail stats (total/active bans per jail)
- SSH probe count (24h window)
- System health: disk, memory, kernel, reboot-pending flag
- Service status: Docker, fail2ban, UFW, custom apps
- SSH tunnel access pattern for when local TUN proxy blocks direct connections

## Installation

Copy the skill directories into your Claude Code skills folder:

**Linux / macOS:**
```bash
cp -r vps-security-audit vps-initial-hardening ~/.claude/skills/
```

**Windows (Git Bash):**
```bash
cp -r vps-security-audit vps-initial-hardening ~/.claude/skills/
```

**Windows (PowerShell):**
```powershell
Copy-Item -Recurse vps-security-audit, vps-initial-hardening "$env:USERPROFILE\.claude\skills\"
```

After copying, the skills are available immediately in the current and future Claude Code sessions.

## Platform Notes

**VPS side (Ubuntu server):** All commands are identical regardless of your local OS — these skills target the remote server.

**Local machine (where you run Claude Code):**

| Task | Windows | Linux / macOS |
|------|---------|---------------|
| SSH tunnel | Xshell local port forward | `ssh -fNL PORT:VPS_IP:22 ...` |
| Key path | `C:\Users\NAME\.ssh\KEY` or `~/.ssh/KEY` in Git Bash | `~/.ssh/KEY` |
| Key generation | `ssh-keygen` in Git Bash / PowerShell / WSL | `ssh-keygen` in terminal |

## Background

These skills were developed while hardening multiple Ubuntu 24.04 VPS instances running ZenithPanel (VLESS+Reality, Hysteria2) against continuous SSH scanning and brute-force attacks. Key findings:

- The custom `sshd-preauth-reset` jail blocks 30–50% more attackers than the default `sshd` jail alone (catches "connection reset [preauth]" from port scanners)
- Ubuntu 24.04 uses socket activation for SSH — `ssh.socket` restart kills all active sessions; never run it while connected
- `50-cloud-init.conf` sets `PasswordAuthentication yes` and wins over `sshd_config` due to alphabetical sort — must be disabled before key-only auth works
- ZenithPanel must use `--network host` Docker networking; it auto-generates a random admin port stored in SQLite

## License

MIT
