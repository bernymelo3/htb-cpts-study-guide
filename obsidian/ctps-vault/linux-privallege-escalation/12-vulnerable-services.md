# Section 12 — Vulnerable Services

## ID
530

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 12 — Vulnerable Services

## Description
Demonstrates privilege escalation by exploiting a vulnerable SUID service (GNU Screen 4.5.0) that allows arbitrary file creation as root via its logfile feature, leading to full root access through ld.so.preload hijacking.

## Tags
privesc, screen, suid, ld.so.preload, vulnerable-services, linux

## Commands
- screen -v
- searchsploit "GNU Screen 4.5.0"
- searchsploit -m linux/local/41154.sh
- scp <EXPLOIT>.sh <USER>@<TARGET_IP>:/home/<USER>/
- ./41154.sh
- cat /root/screen_exploit/flag.txt

## What This Section Covers
Outdated or misconfigured services running with elevated privileges (SUID) can be exploited for privilege escalation. GNU Screen 4.5.0 is a classic example — it runs as SUID root and its `-L` (logfile) flag lets you write to arbitrary files as root, including `/etc/ld.so.preload`, which the dynamic linker reads before every program execution. This turns a simple file-write into full root code execution.

## Methodology
1. Identify the installed Screen version with `screen -v` — confirm it's 4.05.00 (vulnerable).
2. On the attack box, find the public exploit with `searchsploit "GNU Screen 4.5.0"` and mirror `linux/local/41154.sh`.
3. Transfer the exploit to the target via `scp` (or any file transfer method — wget, curl, nc, etc.).
4. Make it executable with `chmod +x 41154.sh` and run it — the script handles compilation and exploitation automatically.
5. Verify root access with `id` then grab the flag.

## Multi-step Workflow (optional)
```
# On attack box
searchsploit -m linux/local/41154.sh
scp 41154.sh htb-student@<TARGET_IP>:/home/htb-student/

# On target
chmod +x 41154.sh
./41154.sh
cat /root/screen_exploit/flag.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Submit the contents of flag.txt in /root/screen_exploit | 91927dad55ffd22825660da88f2f92e0 | Ran Screen 4.5.0 exploit → root shell → cat flag |

## Key Takeaways
- Always check for outdated SUID binaries — `find / -perm -4000 -type f 2>/dev/null` — then cross-reference versions against known exploits.
- The Screen 4.5.0 exploit chain: malicious shared lib → write to `/etc/ld.so.preload` via Screen's logfile feature → next SUID execution loads the lib → lib sets SUID bit on attacker's shell binary.
- `/etc/ld.so.preload` is loaded by the dynamic linker before any shared library for every dynamically linked binary — controlling it means code execution as whatever user runs the next binary.
- `searchsploit` + `searchsploit -m` is the standard workflow for finding and copying known exploits from ExploitDB locally.
- Service-based privesc is about recognizing that a service's elevated privileges (SUID, capabilities, running as root) combined with a bug = root access.

## Gotchas
- The exploit requires `gcc` on the target to compile the shared library and root shell — if gcc is missing, you'd need to cross-compile on your attack box for the target architecture and transfer the binaries.
- The script uses `#!/bin/bash` — make sure bash is available and the file has execute permissions before running.
- The exploit writes to `/etc/ld.so.preload` which is a system-wide file — in a real engagement this is noisy and could break things; clean up after yourself.
