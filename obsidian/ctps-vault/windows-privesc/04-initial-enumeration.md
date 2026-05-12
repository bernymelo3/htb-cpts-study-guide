# NOTE template — hands-on section (has commands)

## ID
602

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 4 — Initial Enumeration

## Description
Covers systematic manual enumeration of a Windows host after gaining initial access: system info, running processes, environment variables, network connections, users, groups, privileges, and installed software to identify privilege escalation paths.

## Tags
initial-enumeration, whoami, systeminfo, netstat, tasklist, windows-privesc

## Commands
- tasklist /svc
- set
- systeminfo
- wmic qfe
- Get-HotFix | ft -AutoSize
- wmic product get name
- netstat -ano
- query user
- echo %USERNAME%
- whoami /priv
- whoami /groups
- net user
- net localgroup
- net localgroup <GROUP_NAME>
- net accounts

## What This Section Covers
After landing on a Windows host, methodical manual enumeration reveals the OS version, patch level, running services, user privileges, group memberships, and listening ports — all of which feed directly into choosing a privilege escalation path. This section walks through each enumeration command and explains what to look for in the output, from non-standard PATH entries and outdated patches to service accounts with dangerous privileges.

## Methodology
1. Check your current user context with `echo %USERNAME%`, then enumerate privileges with `whoami /priv` — look for non-default privileges like `SeTakeOwnershipPrivilege`, `SeImpersonatePrivilege`, `SeDebugPrivilege` that enable direct privesc.
2. Check group memberships with `whoami /groups` — look for membership in privileged groups (Administrators, Backup Operators, etc.).
3. Run `systeminfo` to identify OS version, build number, patch level, and whether it's a VM — cross-reference installed hotfixes with known CVEs.
4. List hotfixes with `wmic qfe` or `Get-HotFix | ft -AutoSize` — if patches are old, kernel exploits may apply.
5. Enumerate running processes with `tasklist /svc` — identify non-standard services (FileZilla, Tomcat, SQL Server) and note services running as SYSTEM.
6. Check environment variables with `set` — look for custom PATH entries (writable directories = DLL injection), HOMEDRIVE pointing to file shares, and roaming profile paths.
7. List installed software with `wmic product get name` — look for vulnerable versions of known software (Java, FileZilla, Putty) and run credential-harvesting tools like LaZagne.
8. Enumerate listening ports with `netstat -ano` — identify services only accessible locally and cross-reference PIDs with `tasklist /svc` or Task Manager to map ports to services.
9. Check logged-in users with `query user` — identify active sessions, idle users, and session types (console vs RDP).
10. Enumerate all users with `net user`, all groups with `net localgroup`, and inspect interesting groups with `net localgroup <GROUP_NAME>` — especially Administrators, Backup Operators, Remote Desktop Users.
11. Review password policy with `net accounts` — check for lockout threshold (brute force feasibility), minimum password length, and password history.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: What non-default privilege does the htb-student user have? | SeTakeOwnershipPrivilege | Open CMD as Administrator → `whoami /priv` → non-default privilege listed (even though state is Disabled) |
| Q2: Who is a member of the Backup Operators group? | sarah | `net localgroup "backup operators"` |
| Q3: What service is listening on port 8080 (service name not the executable)? | Tomcat8 | `netstat -ano` → find PID for port 8080 → Task Manager Details tab → match PID to service name |
| Q4: What user is logged in to the target host? | sccm_svc | `query user` → look for the user that is NOT htb-student |
| Q5: What type of session does this user have? | console | `query user` → SESSIONNAME column for sccm_svc shows "console" |

## Key Takeaways
- A privilege listed as "Disabled" in `whoami /priv` still matters — it means the user HAS the privilege, it's just not currently active. Many privesc techniques (e.g., SeTakeOwnershipPrivilege) can be enabled and exploited.
- `netstat -ano` gives you PIDs, but you need to cross-reference with `tasklist /svc` or Task Manager to get the actual service name — the question specifically asks for the service name, not the executable.
- `query user` reveals session types: `console` means physical/local login (or VM console), `rdp-tcp#N` means RDP — a console session from a service account like `sccm_svc` is suspicious and worth investigating further.
- Always check the Backup Operators group — members can read/write any file on the system regardless of ACLs, which is a direct path to credential theft (SAM, NTDS.dit).
- The `set` command is underrated: custom PATH entries with writable directories enable DLL hijacking, and HOMEDRIVE pointing to a network share may expose sensitive files across the organization.
- `wmic product get name` can be slow but reveals installed software that `tasklist` won't show if the service isn't currently running.

## Gotchas
- Q1 requires running CMD **as Administrator** (right-click → Run as Administrator, supply the password) — a normal CMD shell may not show all privileges.
- Q3 asks for the **service name** (Tomcat8), not the executable name (tomcat8.exe) — read questions carefully.
- `wmic qfe` may not show all hotfixes — some can be hidden from non-admin users. Use `Get-HotFix` in PowerShell as a cross-check.
- `wmic product get name` can take a very long time to execute on systems with many installed programs — be patient or use `Get-WmiObject -Class Win32_Product` in PowerShell for formatted output.
