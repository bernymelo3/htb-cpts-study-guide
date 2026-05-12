# Event Log Readers

## ID
704

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 11 — Event Log Readers

## Description
Leverage Event Log Readers group membership to search Windows Security event logs for credentials exposed in logged command lines (event ID 4688).

## Tags
event-log-readers, credential-hunting, wevtutil, get-winevent, windows

## Commands
- `net localgroup "Event Log Readers"`
- `wevtutil qe Security /rd:true /f:text | Select-String "/user"` (PowerShell — confirmed working)
- `wevtutil qe Security /rd:true /f:text | findstr /i "/user"` (CMD alternative)
- `wevtutil qe Security /rd:true /f:text /r:<REMOTE_HOST> /u:<USER> /p:<PASS> | findstr "/user"`
- `Get-WinEvent -LogName security | where { $_.ID -eq 4688 -and $_.Properties[8].Value -like '*/user*'} | Select-Object @{name='CommandLine';expression={ $_.Properties[8].Value }}`
- `wevtutil qe "Microsoft-Windows-PowerShell/Operational" /rd:true /f:text | findstr /i "<KEYWORD>"`

## What This Section Covers
When process creation auditing is enabled (event ID 4688), Windows logs the full command line of every new process. If users or scripts pass credentials inline (e.g. `net use /user:name password`), those creds end up in the Security event log. Members of the Event Log Readers group can query these logs and harvest plaintext credentials.

## Methodology
1. Confirm group membership: `net localgroup "Event Log Readers"`.
2. Search Security logs for `/user` patterns: `wevtutil qe Security /rd:true /f:text | Select-String "/user"` (in PowerShell; use `findstr /i "/user"` in CMD).
3. Look for `net use`, `runas`, or other commands that passed credentials inline.
4. Also check PowerShell Operational logs for script block logging output — these are accessible without admin rights.
5. For remote hosts, use `wevtutil` with `/r:<HOST> /u:<USER> /p:<PASS>` flags.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Password for user mary | (submit from your output) | `wevtutil qe Security /rd:true /f:text \| findstr /i "/user"` — find mary's entry |

## Key Takeaways
- `wevtutil` works from CMD or PowerShell for Event Log Readers members; `Get-WinEvent` on the Security log requires admin or registry key permission adjustments.
- PowerShell Operational logs (`Microsoft-Windows-PowerShell/Operational`) are accessible to unprivileged users and may also contain credentials if script block or module logging is enabled.
- This is a realistic attack — the section author caught a pentester on a real engagement because process command-line auditing flagged suspicious commands.
- On assessments, always check event logs for creds — admins and scripts frequently pass passwords inline with `net use`, `runas`, `schtasks`, etc.

## Gotchas
- `Get-WinEvent -LogName security` fails for non-admins even if they're in Event Log Readers — use `wevtutil` instead.
- The `/rd:true` flag reads events in reverse chronological order (newest first) — useful for large logs.
- `wevtutil` can query remote hosts with `/r:` — useful for lateral movement scenarios.
