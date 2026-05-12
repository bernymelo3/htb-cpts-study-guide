# Linux Privilege Escalation — Section 1: Introduction to Linux Privilege Escalation

## ID
101

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 1 — Introduction to Linux Privilege Escalation

## Description
Covers the core enumeration checklist for Linux privilege escalation, including system info, user artifacts, permissions, and scheduled tasks — the foundation for finding a path to root.

## Tags
linux, privilege-escalation, enumeration, theory, methodology

## TL;DR — What's Important
- **Enumeration is key:** Manual enumeration skills matter more than any automated script; scripts help but you must know what to look for and why.
- **Kernel exploits are risky:** Public kernel exploits can crash the system; always understand the exploit and avoid production unless you have explicit permission.
- **Sudo with NOPASSWD is a goldmine:** `sudo -l` shows what you can run without a password — some commands (like `tcpdump`, `vim`, `less`) can lead to a full root shell.
- **Check writeable files in cron jobs:** If a cron job runs a world‑writeable script as root, you can modify it to execute arbitrary commands at the next run.
- **Readable shadow file means offline cracking:** Grabbing `/etc/shadow` (or password hashes from `/etc/passwd`) allows you to crack passwords offline, potentially moving to other users or systems.

## Concept Overview
Privilege escalation on Linux is the process of moving from a low‑privileged user (e.g., a webserver user) to the root account, giving full control over the host. This is essential during assessments to access sensitive data, capture traffic, and pivot to other hosts. The most critical phase is enumeration — gathering information about the system, users, running processes, permissions, and scheduled tasks. Automated scripts like LinEnum help, but understanding what each piece of information means is what allows you to spot subtle misconfigurations.

## Key Concepts

### Comparison Table (optional)
| Manual Enumeration | Automated Scripts (LinEnum, LinPEAS) | When to Use |
|--------------------|---------------------------------------|--------------|
| Full control, no detection risk | Faster, but may trigger AV / EDR | Always start manually; use scripts when you need breadth quickly |
| No dependencies | Requires uploading/running a binary | Manual for stealth/constrained envs; scripts for CTF/lab |
| Builds deep understanding | May miss customized configurations | Manual for real engagements where you must be methodical |

### Definitions
- **Privilege Escalation** — Gaining higher access rights than initially granted, e.g., from a standard user to root.
- **SETUID / SETGID** — Permission bits that allow a binary to run with the file owner’s (often root) privileges, regardless of who executes it.
- **Cron Job** — Scheduled task on Linux; if a cron script is writeable by a low‑privileged user, it can be abused for escalation.
- **NOPASSWD** — A sudoers directive allowing a user to run specific commands without entering a password.

### Categories / Types (Enumeration Focus)
- **System Information:** OS version, kernel version, distribution – to identify known public exploits.
- **User & Session Data:** Logged‑in users, home directories, bash history, SSH keys – for lateral movement or credential theft.
- **Permissions & Privileges:** Sudo rights, SETUID binaries, world‑writeable files/directories – direct paths to root.
- **Scheduled & Running Tasks:** Cron jobs, services running as root – opportunities to hijack recurring actions.
- **Secrets & Configs:** Readable shadow file, configuration files with passwords – offline cracking or immediate reuse.

## Why It Matters
During a real penetration test or CTF, you will often land on a low‑privileged shell. Knowing how to systematically enumerate gives you the roadmap to escalate. Many beginners run LinPEAS, get overwhelmed by output, and miss obvious wins like a NOPASSWD sudo entry or a writeable backup script in cron.daily. Mastery of manual enumeration means you can quickly prioritize the most promising leads and avoid wasting time on rabbit holes.

## Defender Perspective
- **Detection:** Unexpected `sudo -l` or `ps aux` commands, access to `/etc/shadow`, or reads of `.bash_history` can be caught by EDR agents that monitor process command lines.
- **Common mitigations:**
    - Restrict `sudo` usage and never use `NOPASSWD` except for truly safe, non‑interactive commands (and even then, be cautious).
    - Monitor cron scripts for unauthorised changes (file integrity monitoring).
    - Store password hashes in a properly protected `/etc/shadow` (not world‑readable).
    - Remove SETUID bits from binaries unless absolutely necessary.
- **MITRE ATT&CK references:**
    - **T1087** – Account Discovery (who is logged in, user home dirs)
    - **T1082** – System Information Discovery (OS, kernel)
    - **T1053.003** – Cron (abusing scheduled tasks)
    - **T1548.001** – SETUID (abusing setuid binaries)

## Key Takeaways
- Manual enumeration is a skill you build over time; start with the checklist in this section and repeat it on every Linux machine you compromise.
- The same enumeration data can lead to different escalation paths: e.g., a cron job running `backup.sh` as root – if you can write `backup.sh`, you win; if you can only write the directory, you might perform a symbolic link attack.
- Kernel exploits are the last resort — they are noisy, unstable, and often fail. Always look for misconfigurations (sudo, cron, writeable files) first.
- SSH keys found during enumeration are not just for the current host; they can give you access to other systems in the network. Always check ARP cache and known_hosts.
- A readable `.bash_history` often contains a treasure trove: passwords on command lines, internal paths, and even commands that reveal how to escape restricted environments.

## Gotchas
- **`sudo -l` shows you can run `vi` or `less` as root** – you can spawn a root shell from those pagers by typing `:!/bin/bash` (or `!/bin/sh`). Many people miss this.
- **Writable `/etc/passwd`** – if it’s world‑writeable, you can add a new root user with a known password. It’s rare but a guaranteed win.
- **NOPASSWD on `tcpdump`** – you can dump packets that may contain credentials, but also you can use `tcpdump`’s `-z` option to execute a script as root. Check GTFOBins.
- **Relative path in cron job** – e.g., `* * * * * backup.sh` instead of `/full/path/backup.sh`. If you can control `$PATH`, you can hijack the command.
- **Forgetting to check `ldd` on SETUID binaries** – a binary that loads a custom library from a world‑writeable directory can be tricked into running arbitrary code as root.