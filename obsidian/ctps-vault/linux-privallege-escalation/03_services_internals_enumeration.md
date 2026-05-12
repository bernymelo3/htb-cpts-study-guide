# Linux Privilege Escalation — Linux Services & Internals Enumeration

## ID
601

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 3 — Linux Services & Internals Enumeration

## Description
Deep enumeration of running services, installed packages, cron jobs, process internals, config files, scripts, and bash history to surface privilege escalation vectors missed in the initial sweep.

## Tags
linux-privesc, services, cron, processes, config-files, internals

## Commands
- `cat /etc/hosts`
- `lastlog`
- `w`
- `history`
- `find / -type f \( -name *_hist -o -name *_history \) -exec ls -l {} \; 2>/dev/null`
- `ls -la /etc/cron.daily/`
- `apt list --installed | tr "/" " " | cut -d" " -f1,3 | sed 's/[0-9]://g' | tee -a installed_pkgs.list`
- `sudo -V`
- `ps aux | grep root`
- `find / -type f \( -name *.conf -o -name *.config \) -exec ls -l {} \; 2>/dev/null`

## What This Section Covers
After the initial environment sweep, this section goes deeper into the host's internals: what services are running (especially as root), what packages are installed (and if any are vulnerable), cron job configurations, bash history of other users, config and script files with hardcoded secrets, and process command lines via `/proc`. Each finding feeds into specific attack paths later in the module.

## Methodology
1. Check **/etc/hosts** for internal hostnames or domain info: `cat /etc/hosts`.
2. Review **login history**: `lastlog` (who logs in and when), `w` (who is currently on), `who`.
3. Read **bash history**: `history` for current user, then `find / -type f \( -name *_hist -o -name *_history \)` to locate all history files — users sometimes type passwords on the command line.
4. Enumerate **cron jobs**:
   - `ls -la /etc/cron.daily/` (also hourly, weekly, monthly)
   - `crontab -l` (current user's crontab)
   - `cat /etc/crontab` (system-wide crontab)
5. Inspect **/proc** for running process command lines: `find /proc -name cmdline -exec cat {} \; 2>/dev/null | tr " " "\n"` — may reveal SSH connections, passwords as arguments.
6. List **installed packages**: `apt list --installed | tr "/" " " | cut -d" " -f1,3 | sed 's/[0-9]://g' | tee -a installed_pkgs.list`.
7. Check **sudo version**: `sudo -V` — versions < 1.8.28 may be vulnerable (CVE-2019-14287); 1.8.31 may be vulnerable to Baron Samedit (CVE-2021-3156).
8. List **binaries**: `ls -l /bin /usr/bin/ /usr/sbin/`.
9. **Cross-reference with GTFOBins**: `for i in $(curl -s https://gtfobins.org/api.json | jq -r '.executables | keys[]'); do if grep -q "$i" installed_pkgs.list; then echo "Check GTFO: $i"; fi; done`.
10. Enumerate **root processes**: `ps aux | grep root` — misconfigured services running as root are high-value targets.
11. Use **strace** on suspicious binaries: `strace <COMMAND>` — trace system calls, look for file reads, network connections, secrets.
12. Search **config files**: `find / -type f \( -name *.conf -o -name *.config \) -exec ls -l {} \; 2>/dev/null`.
13. Search **shell scripts**: `find / -type f -name "*.sh" 2>/dev/null | grep -v "src\|snap\|share"`.
14. Enumerate **running services by user**: `ps aux | grep root` to find scripts/binaries run by root with weak permissions.

## Multi-step Workflow (optional)
```bash
# Full services & internals sweep
cat /etc/hosts
lastlog | grep -v "Never"
w
history

# Cron enumeration
ls -la /etc/cron.daily/ /etc/cron.hourly/ /etc/cron.weekly/ /etc/cron.monthly/ 2>/dev/null
crontab -l 2>/dev/null
cat /etc/crontab

# Package + GTFOBins check
apt list --installed 2>/dev/null | tr "/" " " | cut -d" " -f1,3 | sed 's/[0-9]://g' | tee installed_pkgs.list
sudo -V | head -1

# Root process enumeration
ps aux | grep root

# Config and script search
find / -type f \( -name "*.conf" -o -name "*.config" \) -exec ls -l {} \; 2>/dev/null
find / -type f -name "*.sh" 2>/dev/null | grep -v "src\|snap\|share"

# Proc cmdlines
find /proc -name cmdline -exec cat {} \; 2>/dev/null | tr "\0" " " | tr " " "\n"
```

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| What is the latest Python version that is installed on the target? | *(run on target)* | `python3 --version` or `ls /usr/bin/python*` to see all installed versions |

> **SSH:** `htb-student` / `HTB_@cademy_stdnt!` → 10.129.205.110

## Key Takeaways
- **Cron jobs with weak permissions are a top privesc vector** — always enumerate `/etc/cron.*`, `crontab -l`, and `/etc/crontab`.
- **Bash history of other users** may contain passwords typed as command arguments (e.g., `mysql -u root -pSECRET`).
- **`/proc` filesystem** reveals running process command lines including SSH connections and possibly credentials passed as arguments.
- **Cross-referencing installed packages with GTFOBins** identifies binaries that can be abused if they have SUID bits or sudo entries.
- **`sudo -V`** is critical — sudo version 1.8.31 is specifically vulnerable to Baron Samedit (CVE-2021-3156).
- **`strace`** is a powerful diagnostic tool for tracing system calls; useful for spotting files read, network connections, or secrets accessed by a binary.
- **Config files are world-readable by default** on many Linux systems — even without permissions on the parent folder, you may still read the file.
- **Admin shell scripts** (`find / -name "*.sh"`) often have weak permissions and may contain hardcoded credentials.
- **`lastlog | grep -v "Never"`** quickly shows only users who have actually logged in — identifies active accounts.

## Gotchas
- The GTFOBins one-liner requires `curl`, `jq`, and outbound internet access — on isolated targets, download the JSON beforehand and transfer it.
- `find /proc -name cmdline` output uses null bytes as delimiters — pipe through `tr "\0" " "` for readable output.
- `apt list --installed` is Debian/Ubuntu-specific — on RHEL/CentOS use `rpm -qa`, on Arch use `pacman -Q`.
- `strace` may not be available or may require privileges — check with `which strace`.
