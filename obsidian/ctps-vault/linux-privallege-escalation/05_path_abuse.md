# Linux Privilege Escalation — Path Abuse

## ID
603

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 5 — Path Abuse

## Description
Exploiting misconfigured PATH environment variables to hijack command execution — if a writable or non-default directory appears in PATH, an attacker can replace common binaries with malicious scripts to escalate privileges.

## Tags
linux-privesc, path-abuse, environment, hijacking

## Commands
- `echo $PATH`
- `env | grep PATH`
- `PATH=.:${PATH} && export PATH`
- `touch <BINARY_NAME>`
- `echo '<MALICIOUS_COMMAND>' > <BINARY_NAME>`
- `chmod +x <BINARY_NAME>`

## What This Section Covers
PATH is the environment variable that tells Linux where to look for executables when a command is typed without an absolute path. If a non-default or writable directory is included in a user's PATH (especially `.` — the current directory), an attacker can place a malicious script with the same name as a commonly-used binary, and it will execute instead of the real one. This is especially dangerous when a privileged process (cron job, SUID binary, sudo script) calls commands without absolute paths.

## Methodology
1. Check current PATH: `echo $PATH`.
2. Compare against the Linux default: `/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games` — anything extra is suspect.
3. Check if any PATH directory is **writable** by your user: `ls -la /usr/local/sbin` (or whichever directory looks unusual).
4. If `.` (current working directory) is in PATH, you can hijack commands from wherever you stand.
5. If you can **modify another user's PATH** (via `.bashrc`, `.profile`, or environment files), add `.` to the front: `PATH=.:${PATH}`.
6. Identify a **target command** that a privileged process calls without an absolute path (e.g., a cron job running `ls` instead of `/bin/ls`).
7. Create a malicious script with that command's name in the writable PATH directory:
   ```bash
   touch ls
   echo '/bin/bash -p' > ls   # or a reverse shell
   chmod +x ls
   ```
8. Wait for the privileged process to execute, or trigger it manually.

## Multi-step Workflow (optional)
```bash
# 1. Check current PATH
echo $PATH

# 2. Add current directory to PATH
PATH=.:${PATH}
export PATH

# 3. Create hijack script (example: replace "ls")
touch ls
echo 'echo "PATH HIJACKED — could be a reverse shell"' > ls
chmod +x ls

# 4. Test it
ls
# Output: PATH HIJACKED — could be a reverse shell

# 5. Cleanup
rm ls
```

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Review the PATH of the htb-student user. What non-default directory is part of the user's PATH? | *(run on target)* | `echo $PATH` — compare against default, the extra directory is the answer |

> **SSH:** `htb-student` / `Academy_LLPE!` → 10.129.2.210

## Key Takeaways
- **Default Linux PATH:** `/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games` — memorize this so you can instantly spot additions.
- **`.` in PATH is dangerous** — it means any script in the current directory with the same name as a system command will execute first.
- **PATH abuse is most powerful when combined with cron jobs or SUID binaries** — if a root cron job calls `tar` without `/bin/tar`, you place a malicious `tar` in a writable PATH directory and get root execution.
- **PATH order matters** — directories listed first take priority; prepending `.` or a writable dir ensures your malicious script runs before the real binary.
- **Always check if PATH directories are writable**: `ls -ld /usr/local/sbin` — if your user or group has write access, you can drop scripts there.
- In a real engagement, look for scripts that call commands without absolute paths — these are your PATH abuse targets.

## Gotchas
- PATH abuse **only works if the target process uses relative paths** (calls `ls` not `/bin/ls`) — well-written scripts use absolute paths.
- The malicious script **must be executable** — always `chmod +x` after creating it.
- If you hijack a command like `ls` in your own session, you'll break your own shell — use `/bin/ls` explicitly or clean up immediately.
- Modifying PATH with `PATH=.:${PATH}` only affects the current shell session unless you also `export` it or write it to `.bashrc`/`.profile`.
- Some systems use `secure_path` in `/etc/sudoers` which overrides the user's PATH when running sudo — check with `sudo -l` (look for `env_reset` and `secure_path`).
