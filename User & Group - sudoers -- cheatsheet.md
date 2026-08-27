# Linux User, Group & Sudoers — Command Cheat Sheet
### NovaCore Technologies — `prod-app-01`

Quick reference for every command used across Phase 1 (Users & Groups), Phase 2 (Sudoers), and Phase 3 (Deployment File Ops). Grouped by task, not by phase, so you can jump straight to what you need.

---

## 👥 Groups

| Command | Purpose |
|---|---|
| `sudo groupadd <name>` | Create a new group |
| `sudo groupdel <name>` | Delete a group |
| `getent group <name>` | Show a group's GID and members |
| `getent group \| grep <pattern>` | Search all groups |
| `cat /etc/group` | View raw group database |

```bash
sudo groupadd platform-eng
getent group platform-eng
```

---

## 🧑 Users

| Command | Purpose |
|---|---|
| `sudo useradd -m -s /bin/bash -c "Full Name" <user>` | Create user + home dir + shell + comment |
| `sudo useradd -r -s /usr/sbin/nologin <user>` | Create a system/service account (no login) |
| `sudo adduser <user>` | Interactive user creation (Debian/Ubuntu) |
| `sudo userdel -r <user>` | Delete user **and** their home directory |
| `getent passwd <user>` | Show one user's passwd entry |
| `getent passwd \| grep <pattern>` | Search all users |
| `cat /etc/passwd` | View raw user database |
| `id <user>` | Show UID, GID, and all group memberships |
| `groups <user>` | List a user's groups |

```bash
sudo useradd -m -s /bin/bash -c "Arjun Kumar" arjun.k
id arjun.k
```

---

## 🔑 Passwords & Account Expiry

| Command | Purpose |
|---|---|
| `sudo passwd <user>` | Set/change a user's password |
| `sudo chage -d 0 <user>` | Force password change at next login |
| `sudo chage -l <user>` | View password aging/expiry info |
| `sudo passwd -S <user>` | Show account status (locked/unlocked, `L`/`P`) |
| `sudo usermod -L <user>` | **Lock** an account (disable password login) |
| `sudo usermod -U <user>` | **Unlock** an account |

```bash
sudo passwd arjun.k
sudo chage -d 0 arjun.k
sudo chage -l arjun.k
```

---

## 🔗 Group Membership

| Command | Purpose |
|---|---|
| `sudo usermod -aG <group> <user>` | **Add** user to a group (append — safe) |
| `sudo usermod -G <group1>,<group2> <user>` | **Replace** all supplementary groups (⚠️ overwrites existing) |
| `id <user>` | Verify current group memberships |
| `groups <user>` | Quick list of a user's groups |

```bash
# ✅ Correct — adds without removing existing groups
sudo usermod -aG platform-eng arjun.k

# ⚠️ Careful — this REPLACES all supplementary groups
sudo usermod -G devops student2
```

> **Rule of thumb:** always use `-aG`, never bare `-G`, unless you explicitly intend to wipe existing group memberships.

---

## 🛡️ Sudoers (`/etc/sudoers.d/`)

| Command | Purpose |
|---|---|
| `sudo visudo` | Safely edit the main `/etc/sudoers` file |
| `sudo visudo -f <path>` | Safely edit a specific sudoers drop-in file |
| `sudo visudo -c` | Validate syntax of **all** sudoers files |
| `sudo visudo -c -f <path>` | Validate syntax of **one** file before trusting it |
| `sudo touch /etc/sudoers.d/<name>` | Create a new drop-in sudoers file |
| `sudo chmod 440 /etc/sudoers.d/<name>` | Required permission (sudo refuses writable files) |
| `sudo chown root:root /etc/sudoers.d/<name>` | Required ownership |
| `sudo -l` | Show your own sudo privileges |
| `sudo -l -U <user>` | Show another user's sudo privileges |

```bash
sudo touch /etc/sudoers.d/platform-eng
sudo chmod 440 /etc/sudoers.d/platform-eng
sudo chown root:root /etc/sudoers.d/platform-eng
sudo visudo -f /etc/sudoers.d/platform-eng
sudo visudo -c -f /etc/sudoers.d/platform-eng
```

**Sudoers syntax patterns used:**

```text
%groupname ALL=(ALL:ALL) ALL                          # full sudo for a group
%groupname ALL=(ALL) /path/to/cmd1, /path/to/cmd2      # restricted to specific commands
Cmnd_Alias NAME = /path/to/cmd                         # reusable named command set
%groupname ALL=(ALL) NAME                              # grant using the alias
username ALL=(ALL:ALL) NOPASSWD: ALL                   # passwordless (use sparingly!)
Defaults logfile="/var/log/sudo.log"                   # dedicated sudo audit log
```

---

## 🐧 RHEL/CentOS Equivalent

| Ubuntu/Debian | RHEL/CentOS |
|---|---|
| `sudo` group | `wheel` group |
| `sudo usermod -aG sudo <user>` | `sudo usermod -aG wheel <user>` |
| `getent group sudo` | `getent group wheel` |

---

## 📄 Verification & Auditing

| Command | Purpose |
|---|---|
| `id <user>` | Full UID/GID/group summary |
| `groups <user>` | Group membership only |
| `getent passwd \| grep <user>` | Confirm user exists correctly |
| `getent group \| grep <group>` | Confirm group exists correctly |
| `sudo -l -U <user>` | Confirm exact sudo grants |
| `sudo cat /var/log/sudo.log` | Review sudo command history |
| `sudo tail -f /var/log/sudo.log` | Watch sudo activity live |
| `sudo journalctl -u sudo \| grep <user>` | Search systemd journal for a user's sudo activity |

---

## 📁 File Operations (`cp` / `mv`) — Deployment Context

| Command | Purpose |
|---|---|
| `cp <src> <dst>` | Copy a single file |
| `cp -r <srcdir> <dstdir>` | Copy a directory recursively |
| `cp -v <src> <dst>` | Copy with verbose output (prints each file) |
| `cp -a <src> <dst>` | Copy preserving permissions, ownership, timestamps |
| `cp -av <src> <dst>` | Combine preserve + verbose (recommended for prod copies) |
| `mv <src> <dst>` | Move or rename a file/directory |
| `mv -v <src> <dst>` | Move with verbose output |
| `diff <file1> <file2>` | Compare two files (e.g. old vs. new config) |

```bash
# Timestamped backup before overwrite
sudo cp -av /etc/app/config/app.conf /etc/app/config/app.conf.bak-$(date +%F-%H%M%S)

# Deploy new config
sudo cp -av /opt/releases/app-v2/config/* /etc/app/config/

# Archive rotated logs
sudo mv /var/log/app/app.log /var/log/app/archive/app.log-$(date +%F-%H%M%S)

# Retire old release directory
sudo mv -v /opt/releases/app-v1 /opt/releases/archive/app-v1-$(date +%F-%H%M%S)
```

---

## ⚡ One-Liners Worth Memorizing

```bash
# Create user + set to expire password immediately
sudo useradd -m -s /bin/bash <user> && sudo passwd <user> && sudo chage -d 0 <user>

# Add a user to a group and confirm it in one go
sudo usermod -aG <group> <user> && id <user>

# Validate every sudoers file at once
sudo visudo -c

# Check whether an account is locked
sudo passwd -S <user> | awk '{print $2}'   # "L" = locked, "P" = usable password

# List everyone in a group
getent group <group> | cut -d: -f4
```

---

## 🚫 Common Mistakes to Avoid

| Mistake | Why it's wrong | Correct approach |
|---|---|---|
| Editing `/etc/sudoers` with `nano`/`vim` directly | No syntax check — a typo can lock out **all** sudo access | Always use `visudo` or `visudo -f` |
| `usermod -G` instead of `-aG` | Wipes all existing supplementary groups | Use `-aG` to append |
| Sudoers file left world/group-writable | `sudo` will silently refuse to read it | `chmod 440` + `chown root:root` |
| Granting `%group ALL=(ALL) ALL` when only one command is needed | Violates least privilege | Scope to specific binaries or a `Cmnd_Alias` |
| Bare command names in sudoers (`journalctl` instead of `/usr/bin/journalctl`) | Vulnerable to `$PATH` hijacking | Always use full absolute paths |
| Deploying a human user as the app's file owner | No accountability, wrong permission model | Use a dedicated system account (`useradd -r -s /usr/sbin/nologin`) |
