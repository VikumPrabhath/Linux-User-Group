# Real-World Enterprise Scenario: Linux User & Group Access Management
### Company: **NovaCore Technologies** — Server: `prod-app-01` (Ubuntu 22.04 LTS)

---

## 1. Business Context

NovaCore Technologies is a mid-size SaaS company running its core application stack on a fleet of Linux servers. The IT Infrastructure team has just provisioned a new application server, `prod-app-01`, and needs to onboard three teams with **different levels of access**, following the principle of **least privilege**.

This is the kind of task a Linux Administrator / DevOps Engineer receives in week one of a real job — not a synthetic "create student1, student2" exercise, but a ticket with business justification, security constraints, and an audit trail.

**The Ticket (IT-4521):**
> "Provision `prod-app-01` for the Platform, Security, and Support teams. Each team has different privilege requirements. Enforce group-based sudo access — no individual user sudo entries except for the emergency break-glass account. All access must be auditable and reversible."

---

## 2. Architecture Overview

```
                         prod-app-01 (Ubuntu 22.04)
                         ─────────────────────────
                                   │
        ┌──────────────────────────────────────────────────┐
        │                    /etc/group                     │
        │                                                    │
        │   platform-eng   security-team   support-l1        │
        │   (full sudo)    (audit + read)  (restricted cmds) │
        └──────────────────────────────────────────────────┘
                │                 │                │
        ┌───────┴───────┐ ┌───────┴───────┐ ┌──────┴──────┐
        │ arjun.k       │ │ priya.s        │ │ nadeesha.p  │
        │ platform-eng  │ │ security-team  │ │ support-l1  │
        │ + docker      │ │                │ │             │
        └───────────────┘ └────────────────┘ └─────────────┘

        Break-glass account: svc_emergency (NOPASSWD, logged, disabled by default)
```

**Design principles used:**
- **Group-based privilege, not user-based** — sudoers file references `%groupname`, never individual usernames (except the emergency account, which is explicitly justified and logged).
- **Least privilege** — each team gets only the commands it needs, not blanket `ALL=(ALL) ALL`.
- **Separation of duties** — Security team can *read/audit* but not modify system configs.
- **Traceability** — every privileged action must be attributable to a named human account, never a shared login.

---

## 3. Team Roles & Required Access

| Team | Linux Group | Users | Sudo Scope | Reasoning |
|---|---|---|---|---|
| Platform Engineering | `platform-eng` | arjun.k, dilani.f | Full sudo (`ALL=(ALL) ALL`) | Deploys releases, manages services, restarts servers |
| Security Team | `security-team` | priya.s | Read-only diagnostic commands (`journalctl`, `ausearch`, `fdisk -l`) | Performs audits, must not alter configs |
| Support L1 | `support-l1` | nadeesha.p, kasun.w | Restricted (`systemctl restart nginx`, `systemctl status *`) | Handles basic incident response only |
| Emergency | `svc_emergency` (no group) | — | Passwordless, full sudo | Break-glass account for outages, access logged & alerted |

---

## 4. Task Roadmap — Basic to Professional Level

### **Phase 1 — Foundational (Basic)**
1. Create the three department groups: `platform-eng`, `security-team`, `support-l1`.
2. Create each named user account with a home directory and `/bin/bash` shell (`useradd -m -s /bin/bash`), matching corporate naming convention `firstname.lastinitial`.
3. Set a temporary password for each account and force a password change on first login (`chage -d 0`).
4. Assign each user to their correct department group using `usermod -aG`.
5. Verify group membership with `id` and `groups`.

### **Phase 2 — Privilege Separation (Intermediate)**
6. Edit `/etc/sudoers` **only via `visudo`**, and instead of editing the main file directly, create a dedicated file per team under `/etc/sudoers.d/` (real-world best practice — never crowd the main sudoers file):
   - `/etc/sudoers.d/platform-eng`
   - `/etc/sudoers.d/security-team`
   - `/etc/sudoers.d/support-l1`
7. Grant `platform-eng` full sudo via `%platform-eng ALL=(ALL) ALL`.
8. Grant `security-team` command-restricted sudo for **audit-only** tools (no config edits).
9. Grant `support-l1` sudo access limited to restarting/checking a specific service only.
10. Validate every sudoers file with `visudo -c -f <file>` before saving (syntax check — a real production safeguard).

### **Phase 3 — File Operations in a Real Deployment Flow (cp/mv)**
11. As `platform-eng`, copy a new release config from `/opt/releases/app-v2/config/` to `/etc/app/config/` using `cp -av` (preserve attributes + verbose, since this mirrors a real deployment step).
12. Take a timestamped backup of the current config before overwrite: `cp current.conf current.conf.bak-$(date +%F)`.
13. Move old/rotated logs from `/var/log/app/` into `/var/log/app/archive/` using `mv`, simulating log-rotation housekeeping.
14. Move the entire `/opt/releases/app-v1/` directory to `/opt/releases/archive/` after a successful cutover to v2.

### **Phase 4 — Hardening & Auditability (Advanced / Professional)**
15. Disable direct SSH login for all department accounts except via key-based auth (mention only — full SSH hardening is a separate lab).
16. Configure the **emergency break-glass account** (`svc_emergency`) with `NOPASSWD: ALL`, but keep the account **locked** (`usermod -L`) until an incident requires it, and require a change ticket to unlock it.
17. Enable sudo logging to a dedicated file: add `Defaults logfile="/var/log/sudo.log"` in a sudoers drop-in file, so every privileged command by every group is recorded independent of syslog.
18. Test and document: what happens when `security-team` (read-only) attempts a command outside their scope — capture the `sudo: command not allowed` denial as audit evidence.
19. Simulate an **incorrect sudoers edit** (e.g., missing closing parenthesis) and show how `visudo`'s built-in syntax check prevents you from locking yourself out — versus editing `/etc/sudoers` directly with `nano`, which would break sudo system-wide.
20. Produce a final **access review report**: for each group, list members (`getent group <name>`), their granted commands (`sudo -l -U <user>`), and confirm no user has sudo rights outside their assigned group.

---

## 5. Real-World Justification (for your article)

Explain to readers **why** this is closer to actual enterprise practice than the standard "create student1/student2" tutorial:

- **Groups drive sudo, not individuals** → onboarding/offboarding is a single `usermod`/`deluser` action, not a sudoers edit per person.
- **`/etc/sudoers.d/` per team** → change control is scoped; a mistake in one file doesn't risk the whole sudoers config, and each file maps cleanly to a change ticket or Git-tracked config (many shops manage sudoers via Ansible/Puppet from a repo).
- **Command restriction** → mirrors real audit/compliance requirements (SOC 2, ISO 27001) where "who can do what" must be provably limited.
- **Break-glass account** → standard incident-response pattern in real enterprises; access is rare, logged, and requires justification.
- **`visudo -c`** → prevents the single most common real-world outage cause in this domain: a bad sudoers edit locking out all sudo access on a production box.

---

## 6. What to Build Next (suggested follow-up articles)

- Automating this exact setup with an Ansible playbook (idempotent user/group/sudoers provisioning).
- Centralizing identity with LDAP/FreeIPA instead of local `/etc/passwd` accounts.
- Integrating sudo logs with a SIEM (e.g., shipping `/var/log/sudo.log` to Wazuh/ELK).
