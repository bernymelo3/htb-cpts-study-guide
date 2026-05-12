# Linux Privilege Escalation — Section 6: Wildcard Abuse

## ID
106

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 6 — Wildcard Abuse

## Description
Leveraging wildcard expansion in cron jobs (e.g., tar with `*`) to execute arbitrary commands via `--checkpoint` and `--checkpoint-action` options — a classic privilege escalation vector.

## Tags
linux, privilege-escalation, wildcard, cron, tar, theory

## TL;DR — What's Important
- **Wildcards are expanded by the shell before the command runs:** If a cron job uses `*` (e.g., `tar -zcf backup.tar.gz *`), filenames in the current directory can become command-line arguments.
- **tar’s `--checkpoint` and `--checkpoint-action` options allow arbitrary command execution:** Create files named `--checkpoint=1` and `--checkpoint-action=exec=sh script.sh`. When tar sees them, it executes the command.
- **You must be able to write files in the directory where the cron job runs:** Typically the user's home directory or a world‑writeable location.
- **The command runs with the cron job’s privileges (often root):** Abusing this gives you root access immediately.
- **Watch for relative paths in cron jobs:** If the job uses `cd /some/dir && tar -czf *`, you win if you can write to `/some/dir`.

## Concept Overview
Wildcard characters (`*`, `?`, `[]`) are interpreted by the shell, not by the command itself. In a cron job that uses a wildcard (e.g., `tar -czf backup.tar.gz *`), tar receives a list of all filenames in the current directory as separate arguments. If an attacker can create files with names that look like tar options (e.g., `--checkpoint=1`), those strings are passed to tar as command-line flags. Combined with tar’s `--checkpoint-action=exec=...` feature, this becomes arbitrary command execution at the privilege level of the cron job.

## Key Concepts

### Definitions
- **Wildcard (globbing)** — Shell feature that expands patterns like `*`, `?`, `[]` into matching filenames before executing a command.
- **`tar --checkpoint`** — An option that makes tar emit progress messages every N records.
- **`tar --checkpoint-action`** — An option that runs a specified action (e.g., `exec=command`) each time a checkpoint is reached.
- **Cron job** — Scheduled task; if it runs as root and uses wildcards, it becomes a attack surface.

### Attack Steps (Command Sequence)
1. Write a script that grants you sudo privileges (or adds a root user).
   ```bash
   echo 'echo "htb-student ALL=(root) NOPASSWD: ALL" >> /etc/sudoers' > root.sh