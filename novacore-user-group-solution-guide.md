# NovaCore Technologies — User, Group & Sudo Access Management
## Solution Guide (Companion to the Task Roadmap)

> **Server:** `prod-app-01` — Ubuntu 24.04 LTS
> **Scope:** This guide walks through the exact commands, verification steps, and expected output for onboarding three teams (Platform Engineering, Security, Support L1) with group-based, least-privilege sudo access — plus a bonus phase showing how those accounts are used in a real config-deployment workflow.
>
> Each step below matches the numbered task in the roadmap article, so you can use the two documents side by side: task file = "what to do," this file = "how to do it correctly."

---

## Table of Contents

- [Phase 1 — Foundational: Users & Groups](#phase-1--foundational-users--groups)
- [Phase 2 — Privilege Separation: Sudoers](#phase-2--privilege-separation-sudoers)
- [Phase 3 — File Operations in a Deployment Flow (Bonus)](#phase-3--file-operations-in-a-deployment-flow-bonus)

---

## Phase 1 — Foundational: Users & Groups

### Step 1 — Create the Department Groups

Every team gets its own Linux group. Groups are created *before* users so that each account can be placed into its group immediately at creation time.

```bash
sudo groupadd platform-eng
sudo groupadd security-team
sudo groupadd support-l1
```

**Verify:**

```bash
getent group | grep -E "platform-eng|security-team|support-l1"
```

**Expected output:**

```
platform-eng:x:1001:
security-team:x:1002:
support-l1:x:1003:
```

> 💡 **Note:** GIDs are assigned sequentially by the system starting at the first free ID ≥1000 on Ubuntu. Your exact numbers may differ slightly depending on what else exists on the box — that's normal and not something to "fix."

---

### Step 2 — Create User Accounts

Each user is created with a home directory (`-m`), a Bash shell (`-s`), and a display name (`-c`) matching the person on the ticket.

```bash
sudo useradd -m -s /bin/bash -c "Arjun Kumar" arjun.k
sudo useradd -m -s /bin/bash -c "Dilani Fernando" dilani.f
sudo useradd -m -s /bin/bash -c "Priya Sharma" priya.s
sudo useradd -m -s /bin/bash -c "Nadeesha Perera" nadeesha.p
sudo useradd -m -s /bin/bash -c "Kasun Wijesinghe" kasun.w
```

**Verify:**

```bash
getent passwd | grep -E "arjun.k|dilani.f|priya.s|nadeesha.p|kasun.w"
```

**Expected output:**

```
arjun.k:x:1001:1001:Arjun Kumar:/home/arjun.k:/bin/bash
dilani.f:x:1002:1002:Dilani Fernando:/home/dilani.f:/bin/bash
priya.s:x:1003:1003:Priya Sharma:/home/priya.s:/bin/bash
nadeesha.p:x:1004:1004:Nadeesha Perera:/home/nadeesha.p:/bin/bash
kasun.w:x:1005:1005:Kasun Wijesinghe:/home/kasun.w:/bin/bash
```

> ⚠️ **Real-world gotcha worth knowing:** `useradd -m` (without `-g`) gives each user their own **private group** with the same numeric ID as their UID (e.g. `arjun.k:1001` also owns group `arjun.k:1001`). This is a *separate* GID space from the department groups you made in Step 1 — a UID and a GID can share the same number without conflict, because Linux tracks them independently. Don't confuse a user's private group with their department group; the department membership is *supplementary*, added in Step 4.

---

### Step 3 — Set Temporary Passwords & Force a Change on First Login

New employees should never keep an admin-set password. Set a temporary one, then force it to expire immediately.

```bash
sudo passwd arjun.k
sudo passwd dilani.f
sudo passwd priya.s
sudo passwd nadeesha.p
sudo passwd kasun.w
```

Example interactive prompt:

```
Enter new UNIX password: [Temp@123!]
Retype new UNIX password: [Temp@123!]
passwd: password updated successfully
```

**Force a password change at next login:**

```bash
sudo chage -d 0 arjun.k
sudo chage -d 0 dilani.f
sudo chage -d 0 priya.s
sudo chage -d 0 nadeesha.p
sudo chage -d 0 kasun.w
```

**Verify expiry status:**

```bash
sudo chage -l arjun.k
```

**Expected output:**

```
Last password change                                    : password must be changed
Password expires                                        : password must be changed
Password inactive                                       : password must be changed
Account expires                                         : never
Minimum number of days between password change          : 0
Maximum number of days between password change          : 99999
Number of days of warning before password expires        : 7
```

> 💡 **Why this matters:** `chage -d 0` sets the "last changed" date to the epoch, which Linux interprets as "already expired" — so the user is forced to set their own password the moment they log in. This is standard onboarding practice: the admin never keeps knowledge of the user's real password.

---

### Step 4 — Assign Users to Their Department Groups

Use `usermod -aG` (append to supplementary Groups) — **never** plain `-G` here, since that would *replace* all of a user's existing groups instead of adding to them.

```bash
# Platform Engineering
sudo usermod -aG platform-eng arjun.k
sudo usermod -aG platform-eng dilani.f

# Security Team
sudo usermod -aG security-team priya.s

# Support L1
sudo usermod -aG support-l1 nadeesha.p
sudo usermod -aG support-l1 kasun.w
```

---

### Step 5 — Verify Group Membership

```bash
groups arjun.k
```

**Expected output:**

```
arjun.k : arjun.k platform-eng
```

Repeat for each user, or check them all at once:

```bash
for user in arjun.k dilani.f priya.s nadeesha.p kasun.w; do
    echo -n "$user: "
    groups $user
done
```

---

### Bonus — Generate a User Onboarding Report

A quick script to confirm everything from Phase 1 landed correctly — useful to attach to the change ticket as evidence.

```bash
echo "=== NovaCore Technologies User Report ==="
echo "Generated: $(date)"
echo

for user in arjun.k dilani.f priya.s nadeesha.p kasun.w; do
    echo "--- $user ---"
    echo "Home: $(eval echo ~$user)"
    echo "Shell: $(getent passwd $user | cut -d: -f7)"
    echo "Groups: $(groups $user | cut -d: -f2)"
    echo "Password expires: $(sudo chage -l $user | grep "Password expires" | cut -d: -f2)"
    echo
done
```

**Expected output (excerpt):**

```
=== NovaCore Technologies User Report ===
Generated: Thu Aug 27 14:30:22 UTC 2026

--- arjun.k ---
Home: /home/arjun.k
Shell: /bin/bash
Groups: arjun.k platform-eng
Password expires: never

--- dilani.f ---
Home: /home/dilani.f
Shell: /bin/bash
Groups: dilani.f platform-eng
Password expires: never
...
```

> 💡 `chage -l` shows "Password expires: never" here because that field tracks the *maximum password age* policy, not the forced first-login change you set in Step 3 (that's the separate "must be changed" state shown earlier). Both are true at once — don't read this as contradicting Step 3.

**✅ Phase 1 checkpoint:** 3 groups created, 5 users created with home dirs and shells, temporary passwords set with forced change, correct group memberships confirmed. This is the natural stopping point if your practice is scoped to *users & groups only* — Phase 2 below is what turns those group memberships into actual privilege.

---

## Phase 2 — Privilege Separation: Sudoers

### Step 6 — Create the Sudoers Directory Structure

Real environments never edit the monolithic `/etc/sudoers` file directly for team-specific rules. Instead, each team (or each Ansible/Puppet-managed config) gets its own file under `/etc/sudoers.d/`, so a mistake in one file can't corrupt another team's access.

```bash
sudo touch /etc/sudoers.d/platform-eng
sudo chmod 440 /etc/sudoers.d/platform-eng
sudo chown root:root /etc/sudoers.d/platform-eng

sudo touch /etc/sudoers.d/security-team
sudo chmod 440 /etc/sudoers.d/security-team
sudo chown root:root /etc/sudoers.d/security-team

sudo touch /etc/sudoers.d/support-l1
sudo chmod 440 /etc/sudoers.d/support-l1
sudo chown root:root /etc/sudoers.d/support-l1
```

**Verify:**

```bash
ls -la /etc/sudoers.d/
```

**Expected output:**

```
total 12
drwxr-xr-x   2 root root 4096 Aug 27 14:30 .
drwxr-xr-x 132 root root 12288 Aug 27 14:30 ..
-r--r-----   1 root root    0 Aug 27 14:30 platform-eng
-r--r-----   1 root root    0 Aug 27 14:30 security-team
-r--r-----   1 root root    0 Aug 27 14:30 support-l1
-r--r-----   1 root root  958 Aug 27 14:30 README
```

> ⚠️ **Permissions matter here.** `sudo` refuses to read any file in `/etc/sudoers.d/` that is group- or world-writable — it's a deliberate safety check to stop a misconfigured file from becoming a privilege-escalation path. Always `chmod 440` and `chown root:root` every file you create in this directory.

---

### Step 7 — Configure Platform Engineering: Full Sudo

```bash
sudo visudo -f /etc/sudoers.d/platform-eng
```

Add:

```
# NovaCore Technologies - Platform Engineering Team
# Full sudo access for deployment and system management
# Last modified: 2026-08-27

%platform-eng ALL=(ALL:ALL) ALL
```

**Validate syntax before trusting the file:**

```bash
sudo visudo -c -f /etc/sudoers.d/platform-eng
```

**Expected output:**

```
/etc/sudoers.d/platform-eng: parsed OK
```

> 💡 The `%` prefix means "this is a group, not a username." `%platform-eng ALL=(ALL:ALL) ALL` reads as: *any member of the platform-eng group, on any host, may run any command as any user/group.*

---

### Step 8 — Configure Security Team: Read-Only Access

```bash
sudo visudo -f /etc/sudoers.d/security-team
```

Add:

```
# NovaCore Technologies - Security Team
# Read-only diagnostic and audit commands only
# Last modified: 2026-08-27

%security-team ALL=(ALL) /usr/bin/journalctl, /usr/sbin/ausearch, /usr/bin/fdisk -l
```

**Validate:**

```bash
sudo visudo -c -f /etc/sudoers.d/security-team
```

**Expected output:**

```
/etc/sudoers.d/security-team: parsed OK
```

> 💡 Notice this line has **no** `NOPASSWD` and lists exact binary paths, not bare command names. Always use full paths (`/usr/bin/journalctl`, not `journalctl`) in sudoers — otherwise a malicious `journalctl` earlier in the user's `$PATH` could be executed instead of the real one.

---

### Step 9 — Configure Support L1: Service-Specific Access

This team should only be able to restart/check one specific service — not "any service," which is why we scope the wildcard carefully.

```bash
sudo visudo -f /etc/sudoers.d/support-l1
```

Add:

```
# NovaCore Technologies - L1 Support Team
# Using command aliases for better maintainability
# Last modified: 2026-08-27

Cmnd_Alias SERVICE_RESTART = /usr/bin/systemctl restart nginx
Cmnd_Alias SERVICE_STATUS  = /usr/bin/systemctl status *

%support-l1 ALL=(ALL) SERVICE_RESTART, SERVICE_STATUS
```

**Validate:**

```bash
sudo visudo -c -f /etc/sudoers.d/support-l1
```

**Expected output:**

```
/etc/sudoers.d/support-l1: parsed OK
```

> 💡 `Cmnd_Alias` is the sudoers equivalent of a named constant — it keeps the actual grant line (`%support-l1 ALL=(ALL) ...`) short and self-documenting, and makes it trivial to add a second allowed service later without touching the grant itself.

---

### Bonus — Emergency Break-Glass Account

A single, tightly controlled account for outages when normal access isn't enough. It exists, but it's **locked** until someone deliberately unlocks it against a change ticket.

```bash
# Create the account
sudo useradd -m -s /bin/bash -c "Emergency Break-Glass Account" svc_emergency
sudo passwd svc_emergency

# Create its sudoers file
sudo touch /etc/sudoers.d/emergency
sudo chmod 440 /etc/sudoers.d/emergency
sudo chown root:root /etc/sudoers.d/emergency
sudo visudo -f /etc/sudoers.d/emergency
```

Add:

```
# NovaCore Technologies - Emergency Break-Glass Account
# Passwordless full sudo - LOCKED BY DEFAULT
# Only unlock during emergencies with change ticket approval
# Last modified: 2026-08-27

svc_emergency ALL=(ALL:ALL) NOPASSWD: ALL
```

**Validate, then immediately lock the account:**

```bash
sudo visudo -c -f /etc/sudoers.d/emergency
sudo usermod -L svc_emergency
```

**Verify it's locked:**

```bash
sudo passwd -S svc_emergency
```

```
svc_emergency L 08/27/2026 0 99999 7 -1
```

**To use it during a real incident (with a ticket reference), unlock it:**

```bash
sudo usermod -U svc_emergency
```

**And afterward, review who used it:**

```bash
sudo journalctl -u sudo | grep svc_emergency
```

> ⚠️ This is the *only* individually-named entry in the whole sudoers setup, and that's intentional — a single, rare, heavily-logged account is easier to audit than "everyone gets emergency access."

---

### Bonus — Enable Sudo Logging

By default, sudo activity goes to the system auth log mixed in with everything else. A dedicated log file makes audits much faster.

```bash
sudo touch /etc/sudoers.d/logging
sudo chmod 440 /etc/sudoers.d/logging
sudo chown root:root /etc/sudoers.d/logging
sudo visudo -f /etc/sudoers.d/logging
```

Add:

```
# NovaCore Technologies - Sudo Logging Configuration
# All sudo commands are logged to a dedicated file
# Last modified: 2026-08-27

Defaults logfile="/var/log/sudo.log"
Defaults log_year, log_host, syslog=auth
```

```bash
sudo visudo -c -f /etc/sudoers.d/logging
sudo touch /var/log/sudo.log
sudo chmod 640 /var/log/sudo.log
sudo chown root:root /var/log/sudo.log
```

**Validate every sudoers file at once:**

```bash
sudo visudo -c
```

**Expected output:**

```
/etc/sudoers: parsed OK
/etc/sudoers.d/platform-eng: parsed OK
/etc/sudoers.d/security-team: parsed OK
/etc/sudoers.d/support-l1: parsed OK
/etc/sudoers.d/emergency: parsed OK
/etc/sudoers.d/logging: parsed OK
```

---

### Bonus — Test Every User's Sudo Permissions

```bash
cat << 'EOF' > /tmp/test_sudo.sh
#!/bin/bash
echo "=== Testing Sudo Permissions ==="
echo
for user in arjun.k dilani.f priya.s nadeesha.p kasun.w; do
    echo "--- $user ---"
    sudo -l -U $user
    echo
done
EOF
chmod +x /tmp/test_sudo.sh
sudo /tmp/test_sudo.sh
```

**Expected output (excerpt):**

```
=== Testing Sudo Permissions ===

--- arjun.k ---
Matching Defaults entries for arjun.k on prod-app-01:
    logfile=/var/log/sudo.log, log_year, log_host, syslog=auth

User arjun.k may run the following commands on prod-app-01:
    (ALL : ALL) ALL

--- dilani.f ---
...
```

---

### Bonus — Test Actual Command Execution (Allowed vs Denied)

This is the step that actually proves least-privilege is working, not just configured.

```bash
cat << 'EOF' > /tmp/exec_test.sh
#!/bin/bash
echo "=== Testing Command Execution ==="

echo "Platform engineer (arjun.k) - full access:"
sudo -u arjun.k sudo systemctl status nginx
echo "Exit code: $?"
echo

echo "Security team (priya.s) - allowed (read-only):"
sudo -u priya.s sudo journalctl -n 1
echo "Exit code: $?"
echo

echo "Security team (priya.s) - disallowed (config change):"
sudo -u priya.s sudo systemctl restart nginx
echo "Exit code: $?"
echo

echo "Support L1 (nadeesha.p) - allowed (nginx only):"
sudo -u nadeesha.p sudo systemctl status nginx
echo "Exit code: $?"
echo

echo "Support L1 (nadeesha.p) - disallowed (wrong service):"
sudo -u nadeesha.p sudo systemctl restart mysql
echo "Exit code: $?"
EOF
chmod +x /tmp/exec_test.sh
sudo /tmp/exec_test.sh
```

**Expected output (excerpt):**

```
=== Testing Command Execution ===
Platform engineer (arjun.k) - full access:
● nginx.service - A high performance web server...
   Active: active (running) since Thu 2026-08-27 14:30:22 UTC; 1h ago
...
Exit code: 0

Security team (priya.s) - allowed (read-only):
-- Logs begin at Thu 2026-08-27 14:30:22 UTC, end at Thu 2026-08-27 15:30:22 UTC. --
...
Exit code: 0

Security team (priya.s) - disallowed (config change):
Sorry, user priya.s is not allowed to execute '/usr/bin/systemctl restart nginx' as root on prod-app-01.
Exit code: 1
```

> 💡 **Exit code 1 here is success, not failure** — it proves the restriction is actually enforced, which is the entire point of this test.

---

### Bonus — Review the Sudo Log & Build an Audit Report

```bash
sudo cat /var/log/sudo.log
```

**Expected output:**

```
Aug 27 14:30:22 : arjun.k : TTY=pts/0 ; PWD=/home/arjun.k ; USER=root ; COMMAND=/usr/bin/systemctl status nginx
Aug 27 14:31:45 : priya.s : TTY=pts/0 ; PWD=/home/priya.s ; USER=root ; COMMAND=/usr/bin/journalctl -n 1
Aug 27 14:32:10 : priya.s : TTY=pts/0 ; PWD=/home/priya.s ; USER=root ; COMMAND=/usr/bin/systemctl restart nginx
```

Monitor it live:

```bash
sudo tail -f /var/log/sudo.log
```

A single script that pulls everything together into one auditable report:

```bash
cat << 'EOF' > /tmp/final_audit_report.sh
#!/bin/bash
echo "========================================="
echo "NovaCore Technologies - Access Audit Report"
echo "Server: prod-app-01 (Ubuntu 22.04 LTS)"
echo "Generated: $(date)"
echo "========================================="
echo

echo "--- Group Memberships ---"
for group in platform-eng security-team support-l1; do
    echo "Group: $group"
    getent group $group | cut -d: -f4
    echo
done

echo "--- Sudo Permissions ---"
for user in arjun.k dilani.f priya.s nadeesha.p kasun.w; do
    echo "User: $user"
    sudo -l -U $user | grep -v "Matching Defaults"
    echo
done

echo "--- Emergency Account Status ---"
if passwd -S svc_emergency | grep -q " L "; then
    echo "Emergency account: LOCKED ✅"
else
    echo "Emergency account: UNLOCKED ⚠️"
fi
echo

echo "--- Sudo Logging Status ---"
if [ -f "/var/log/sudo.log" ]; then
    echo "Sudo log: ENABLED"
    echo "Log file: /var/log/sudo.log"
    echo "Log size: $(du -h /var/log/sudo.log | cut -f1)"
    echo "Last entry: $(tail -n 1 /var/log/sudo.log)"
else
    echo "Sudo log: NOT FOUND ❌"
fi
echo

echo "--- Sudoers Files Validated ---"
for file in /etc/sudoers.d/*; do
    echo "Checking $file..."
    if visudo -c -f "$file" > /dev/null 2>&1; then
        echo "✅ OK"
    else
        echo "❌ ERROR"
    fi
done
echo

echo "========================================="
echo "Audit Complete"
echo "========================================="
EOF
chmod +x /tmp/final_audit_report.sh
sudo /tmp/final_audit_report.sh
```

**✅ Phase 2 checkpoint:** every team has a dedicated, validated sudoers file scoped to exactly what their job requires; the emergency account exists but is locked; every sudo action is logged to its own file; and you have a repeatable script that proves all of this on demand — which is exactly what an auditor or a "prove it" ticket will ask for.

---

## Phase 3 — File Operations in a Deployment Flow (Bonus)

> This phase goes beyond pure user/group administration — it shows the accounts and privileges from Phases 1–2 actually being *used* in a realistic config-deployment scenario (`cp`/`mv` practice in context). Treat it as an optional extension once the core user & group work above is solid; skip it if you want to keep the exercise scoped strictly to identity/access management.

### Step 10 — Prepare the Directory Structure

```bash
sudo mkdir -p /opt/releases/app-v1/config
sudo mkdir -p /opt/releases/app-v2/config
sudo mkdir -p /opt/releases/archive
sudo mkdir -p /etc/app/config
sudo mkdir -p /var/log/app/archive

# The application runs as a dedicated service account, never as a human user
sudo useradd -r -s /usr/sbin/nologin appuser

sudo chown -R appuser:appuser /opt/releases/ /etc/app/ /var/log/app/
sudo chmod 755 /opt/releases/ /etc/app/config/ /var/log/app/
```

> 💡 `useradd -r` creates a **system account** (no login shell, no password, UID below 1000) — the correct pattern for an account that only exists to own files/processes for an application, never for a human to log into.

Create sample configs for v1 (`/opt/releases/app-v1/config/app.conf`, `database.conf`, `logging.conf`) and v2 (same three files with updated values — new DB host, larger pool size, `DEBUG` log level, etc.). Then simulate the currently-running v1 config being in place:

```bash
sudo cp /opt/releases/app-v1/config/* /etc/app/config/
```

---

### Step 11 — Timestamped Backup, Then Deploy v2

Always back up before overwriting a live config:

```bash
sudo cp /etc/app/config/app.conf /etc/app/config/app.conf.bak-$(date +%F)
sudo cp /etc/app/config/database.conf /etc/app/config/database.conf.bak-$(date +%F)
```

For the real cutover, use verbose + attribute-preserving copies and back up the whole directory, not just individual files:

```bash
sudo cp -av /etc/app/config/app.conf /etc/app/config/app.conf.bak-$(date +%F-%H%M%S)
sudo cp -av /etc/app/config /etc/app/config-backup-$(date +%F-%H%M%S)

sudo cp -av /opt/releases/app-v2/config/* /etc/app/config/
```

**Expected output:**

```
'/opt/releases/app-v2/config/app.conf' -> '/etc/app/config/app.conf'
'/opt/releases/app-v2/config/database.conf' -> '/etc/app/config/database.conf'
'/opt/releases/app-v2/config/logging.conf' -> '/etc/app/config/logging.conf'
```

**Confirm the new version is live:**

```bash
sudo grep -E "version|feature_flags" /etc/app/config/app.conf
```

```
version = 2.0.0
feature_flags = new_dashboard, performance_metrics
```

**Diff old vs. new for the change record:**

```bash
sudo diff /etc/app/config/app.conf.bak-<timestamp> /etc/app/config/app.conf
```

> 💡 `-a` preserves permissions, ownership, and timestamps on copy; `-v` prints every file as it's copied. Both matter for a production deployment: you want the copied file to behave identically to the original, and you want a printed trail of exactly what moved, for the change ticket.

---

### Step 12 — Move Rotated Logs and Retire the Old Release

Archive the current logs before they're overwritten by the new version:

```bash
sudo mv /var/log/app/app.log /var/log/app/archive/app.log-$(date +%F-%H%M%S)
sudo mv /var/log/app/access.log /var/log/app/archive/access.log-$(date +%F-%H%M%S)
```

Retire the old release directory now that v2 is confirmed live:

```bash
sudo mv -v /opt/releases/app-v1 /opt/releases/archive/app-v1-$(date +%F-%H%M%S)
```

**Expected output:**

```
renamed '/opt/releases/app-v1' -> '/opt/releases/archive/app-v1-2026-08-27-144522'
```

> 💡 Note that `mv` here is doing the work of both "rename" and "relocate" at once, since source and destination are on the same filesystem — this is why `mv` is instant even for large directories, while `cp -r` of the same directory would take much longer (it duplicates every byte).

---

### Bonus — Rollback Script

A one-command way to undo a bad v2 deployment:

```bash
sudo bash -c 'cat > /opt/releases/rollback-v2.sh << "EOF"
#!/bin/bash
TIMESTAMP=$(date +%F-%H%M%S)
BACKUP_DIR=/opt/releases/archive

echo "=== Rolling back to v1 ==="
systemctl stop novacore-app
cp -av /etc/app/config/app.conf.bak-$TIMESTAMP /etc/app/config/app.conf
mv $BACKUP_DIR/app-v1-* /opt/releases/app-v1
systemctl start novacore-app
curl -s http://localhost:8080/health
echo "=== Rollback Complete ==="
EOF'
sudo chmod +x /opt/releases/rollback-v2.sh
```

### Bonus — Deployment Verification Script

```bash
sudo bash -c 'cat > /opt/releases/verify-deployment.sh << "EOF"
#!/bin/bash
echo "========================================="
echo "NovaCore Technologies - Deployment Verification"
echo "Timestamp: $(date)"
echo "========================================="

echo "--- Application Status ---"
systemctl is-active novacore-app 2>/dev/null || echo "service not installed"

echo "--- Active Configuration ---"
sudo ls -la /etc/app/config/ | grep -v backup

echo "--- Config Version ---"
sudo grep -E "version|feature_flags" /etc/app/config/app.conf

echo "--- Current Releases ---"
sudo ls -la /opt/releases/

echo "--- Archived Releases ---"
sudo ls -la /opt/releases/archive/

echo "--- Config Backups ---"
sudo ls -la /etc/app/config/*.bak-* 2>/dev/null || echo "No backups found"

echo "--- Logs ---"
sudo ls -la /var/log/app/ /var/log/app/archive/ 2>/dev/null

echo "--- Disk Usage ---"
sudo du -sh /opt/releases/
echo "========================================="
echo "Verification Complete"
echo "========================================="
EOF'
sudo chmod +x /opt/releases/verify-deployment.sh
sudo /opt/releases/verify-deployment.sh
```

**✅ Phase 3 checkpoint:** the account permissions built in Phases 1–2 are now exercised in a realistic deploy → backup → cutover → archive → rollback cycle, with scripts that leave an audit trail behind.

---

## Quick Reference — What Each Team Can Do

| Team | Group | Sudo Scope | Notes |
|---|---|---|---|
| Platform Engineering | `platform-eng` | Full sudo | arjun.k, dilani.f |
| Security | `security-team` | `journalctl`, `ausearch`, `fdisk -l` only | priya.s — read/audit only, no config changes |
| Support L1 | `support-l1` | `systemctl restart nginx`, `systemctl status *` | nadeesha.p, kasun.w — one service only |
| Emergency | *(no group — named account)* | Full, passwordless | `svc_emergency` — locked by default |
