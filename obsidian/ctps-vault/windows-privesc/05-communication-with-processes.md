# NOTE template — hands-on section (has commands)

## ID
603

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 5 — Communication with Processes

## Description
Covers how processes communicate via network sockets and named pipes, and how to enumerate both to find privilege escalation paths — from localhost-only services to insecure named pipe DACLs.

## Tags
named-pipes, netstat, accesschk, network-services, process-communication, windows-privesc

## Commands
- netstat -ano
- tasklist /svc
- tasklist | findstr /c:"<PID>"
- pipelist.exe /accepteula
- gci \\.\pipe\
- accesschk.exe /accepteula \\.\Pipe\<PIPE_NAME> -v
- accesschk.exe -accepteula -w \pipe\* -v
- accesschk.exe -accepteula -w \pipe\<PIPE_NAME> -v

## What This Section Covers
Processes communicate through network sockets and named pipes, and both are attack surfaces for privilege escalation. A service listening only on localhost (127.0.0.1) is often left insecure because developers assume it's not network-accessible. Named pipes with overly permissive DACLs (e.g., Everyone with FILE_ALL_ACCESS or WRITE_DAC) can be abused to escalate to the privilege level of the pipe's owning process — often SYSTEM.

## Methodology
1. Run `netstat -ano` to list all listening ports and established connections.
2. Focus on entries bound to `127.0.0.1` or `::1` (localhost-only services) — these are often insecure and exploitable once you have local access.
3. Cross-reference PIDs from netstat with `tasklist /svc` or `tasklist | findstr /c:"<PID>"` to identify which service owns which port.
4. Enumerate named pipes with `pipelist.exe /accepteula` (Sysinternals) or `gci \\.\pipe\` (PowerShell).
5. Check permissions on interesting named pipes with `accesschk.exe -accepteula -w \pipe\* -v` to find pipes writable by your user or Everyone.
6. Review the DACL output — look for `FILE_ALL_ACCESS`, `WRITE_DAC`, or `FILE_WRITE_DATA` granted to non-admin principals (Everyone, Authenticated Users, your user).
7. If a named pipe has overly permissive DACLs, research the owning service for known privilege escalation exploits.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: What service is listening on 0.0.0.0:21? (two words) | FileZilla Server | `netstat -ano` → find PID for port 21 → `tasklist \| findstr /c:"<PID>"` |
| Q2: Which account has WRITE_DAC privileges over \pipe\SQLLocal\SQLEXPRESS01? | NT SERVICE\MSSQL$SQLEXPRESS01 | `cd C:\Tools\AccessChk` → `accesschk.exe -accepteula -w \pipe\SQLLocal\SQLEXPRESS01 -v` → look for WRITE_DAC in output |

## Key Takeaways
- Localhost-only listeners (127.0.0.1, ::1) are prime targets — they're often unauthenticated because developers assumed network inaccessibility. FileZilla's admin interface on port 14147 is a classic example: connect locally to extract FTP creds or create a root FTP share.
- Named pipes are in-memory files used for IPC — Cobalt Strike uses them for every command, so finding unexpected named pipes (especially mojo-style names without Chrome installed) can indicate red team activity.
- `WRITE_DAC` on a named pipe means you can modify the pipe's access control list — potentially granting yourself full access and escalating to the service account's privileges.
- `FILE_ALL_ACCESS` granted to Everyone on a named pipe is a critical misconfiguration — it means any authenticated user can fully control the pipe (read the WindscribeService CVE as a reference).
- The Splunk Universal Forwarder is a common real-world privesc vector: default install has no authentication and runs as SYSTEM, allowing anyone to deploy applications (code execution).
- Erlang-based services (RabbitMQ, CouchDB, SolarWinds) expose port 25672 for cluster communication — the default cookie is often weak or stored in a readable config file.

## Gotchas
- `accesschk.exe` needs to be run from its directory (e.g., `C:\Tools\AccessChk`) or have its path in PATH — it's not a built-in Windows tool.
- The `-w` flag in accesschk filters for write access — without it you see all permissions, with it you only see pipes your user can write to.
- `pipelist.exe` requires the `/accepteula` flag on first run to suppress the Sysinternals EULA dialog — same for `accesschk.exe` with `-accepteula`.
- Named pipe paths use `\\.\pipe\` notation — don't confuse with UNC paths (`\\server\share`).
