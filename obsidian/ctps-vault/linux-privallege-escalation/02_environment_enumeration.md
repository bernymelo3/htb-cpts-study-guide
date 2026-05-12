# Linux Privilege Escalation — Environment Enumeration

## ID
600

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 2 — Environment Enumeration

## Description
First-touch enumeration after landing on a Linux host: OS, kernel, users, groups, network, drives, hidden files, temp directories, and defenses — the foundation for every privilege escalation path.

## Tags
linux-privesc, enumeration, environment, kernel, users, network

## Commands
- `whoami && id`
- `cat /etc/os-release`
- `uname -a`
- `sudo -l`
- `grep "sh$" /etc/passwd`
- `echo $PATH`
- `ip a`
- `lsblk`
- `find / -type f -name ".*" -exec ls -l {} \; 2>/dev/null`
- `ls -l /tmp /var/tmp /dev/shm`

## What This Section Covers
After gaining initial shell access (e.g., via an unrestricted file upload RCE), you must systematically enumerate the host before attempting any privilege escalation. This section teaches the manual methodology: identify OS/kernel, map users and groups, check network topology, inspect drives and mounts, locate hidden files, and note active defenses.

## Methodology
1. Run the **first five commands** on every new shell: `whoami`, `id`, `hostname`, `ip a`, `sudo -l` — screenshot these for the report.
2. Identify **OS and kernel version**: `cat /etc/os-release` and `uname -a` — check Ubuntu release cycle or distro EOL status; search for public kernel exploits.
3. Check **PATH**: `echo $PATH` — note it down; misconfigured PATH is a privesc vector (covered in Section 5).
4. Dump **environment variables**: `env` — look for leaked passwords or tokens.
5. List **login shells**: `cat /etc/shells` — note if tmux/screen are available.
6. Enumerate **users**: `cat /etc/passwd`, `grep "sh$" /etc/passwd` (users with login shells), `cat /etc/group`, `getent group sudo`.
7. List **home directories**: `ls /home` — plan to enumerate each one for sensitive data.
8. Map **network**: `ip a` (additional NICs = pivot), `route` or `netstat -rn`, `arp -a`, `cat /etc/resolv.conf` (internal DNS = possible AD environment).
9. Inspect **drives and file systems**: `lsblk`, `cat /etc/fstab` (grep for password/credential), `df -h`, unmounted FS: `cat /etc/fstab | grep -v "#" | column -t`.
10. Check **printers**: `lpstat` — queued jobs may contain sensitive data.
11. Note **defenses**: Exec Shield, iptables, AppArmor, SELinux, Fail2ban, Snort, ufw.
12. Find **hidden files**: `find / -type f -name ".*" -exec ls -l {} \; 2>/dev/null | grep <USERNAME>`.
13. Find **hidden directories**: `find / -type d -name ".*" -ls 2>/dev/null`.
14. Check **temp directories**: `ls -l /tmp /var/tmp /dev/shm`.

## Multi-step Workflow (optional)
```bash
# Quick situational awareness one-liner
whoami; id; hostname; ip a | grep inet; sudo -l 2>/dev/null

# Full user enumeration
cat /etc/passwd | cut -f1 -d:
grep "sh$" /etc/passwd
getent group sudo
ls /home

# Network snapshot
ip a; route; arp -a; cat /etc/resolv.conf

# Hidden file sweep
find / -type f -name ".*" -exec ls -l {} \; 2>/dev/null
find / -type d -name ".*" -ls 2>/dev/null
```

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Enumerate the Linux environment and look for interesting files that might contain sensitive data. Submit the flag. | *(flag string from .sh file)* | `find / -name "*.sh" 2>/dev/null \| xargs cat \| grep "HTB"` |

> **SSH:** `htb-student` / `HTB_@cademy_stdnt!` → 10.129.205.110

## Key Takeaways
- **`sudo -l` is the single highest-value command** — can reveal instant root if misconfigured.
- **Password hashes can appear in `/etc/passwd`** (not just `/etc/shadow`) — especially on embedded devices and routers; always check.
- **`grep "sh$" /etc/passwd`** filters only users with real login shells — these are your targets.
- **Additional NICs** (`ip a`) mean potential pivoting into otherwise unreachable subnets.
- **Internal DNS** in `/etc/resolv.conf` suggests an Active Directory environment — changes your enumeration approach.
- **`/tmp` clears on reboot; `/var/tmp` retains ~30 days** — both world-readable, may contain sensitive leftovers.
- **Unmounted drives** may contain sensitive files — if you can escalate to root, mount and explore them.
- Always screenshot the first five commands for the client report.

## Gotchas
- `find / -name *.sh` without quoting the glob can silently fail if `.sh` files exist in the current directory — always quote: `find / -name "*.sh"`.
- `net-tools` package (`ifconfig`, `netstat`) is not always installed — use `ip a` and `ss` as drop-in replacements.
- Hash algorithm identification from `/etc/passwd` or `/etc/shadow`: `$1$` = MD5, `$5$` = SHA-256, `$6$` = SHA-512, `$2a$` = BCrypt, `$7$` = Scrypt, `$argon2i$` = Argon2.
- Ubuntu "Focal Fossa" (20.04) EOL is April 2030 — likely patched, so don't waste time on kernel exploits without evidence.
