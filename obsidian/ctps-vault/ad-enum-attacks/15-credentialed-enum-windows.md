# NOTE — Credentialed Enumeration - from Windows

## ID
550

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 15 — Credentialed Enumeration - from Windows

## Description
Enumerates an AD domain from a Windows attack host using the ActiveDirectory PowerShell module, PowerView, SharpView, Snaffler, and BloodHound/SharpHound to map users, groups, trusts, shares, and Kerberoastable accounts.

## Tags
active-directory, powerview, bloodhound, sharphound, snaffler, windows-enumeration

## Commands
- `Get-Module` / `Import-Module ActiveDirectory`
- `Get-ADDomain`
- `Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName`
- `Get-ADTrust -Filter *`
- `Get-ADGroup -Filter * | select name`
- `Get-ADGroupMember -Identity "<GROUP_NAME>"`
- `Get-DomainUser -Identity <USERNAME> -Domain <DOMAIN> | Select-Object -Property name,samaccountname,description,memberof,whencreated,pwdlastset,lastlogontimestamp,accountexpires,admincount,userprincipalname,serviceprincipalname,useraccountcontrol`
- `Get-DomainGroupMember -Identity "Domain Admins" -Recurse`
- `Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName`
- `Test-AdminAccess -ComputerName <HOST>`
- `.\SharpHound.exe -c All --zipfilename <NAME>`
- `.\Snaffler.exe -d <DOMAIN> -s -v data -o snaffler.log`

## What This Section Covers
Performing credentialed AD enumeration from a Windows host using built-in tools (ActiveDirectory PS module), offensive tools (PowerView, SharpView, Snaffler), and BloodHound for relationship mapping. The goal is to identify misconfigurations, trust relationships, privileged group memberships, Kerberoastable accounts, and sensitive data in file shares that enable lateral/vertical movement.

## Methodology
1. Check available modules with `Get-Module`, then `Import-Module ActiveDirectory` if needed.
2. Enumerate domain basics with `Get-ADDomain` — note child domains, domain functional level, DCs, and domain SID.
3. Find Kerberoastable accounts with `Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName`.
4. Map trust relationships with `Get-ADTrust -Filter *` — note direction, intra-forest vs. external, and transitive status.
5. Enumerate groups with `Get-ADGroup` and drill into high-value groups (Domain Admins, Backup Operators, etc.) with `Get-ADGroupMember`.
6. Switch to PowerView for deeper enumeration: `Get-DomainUser` for detailed user info, `Get-DomainGroupMember -Recurse` for nested group membership, `Get-DomainTrustMapping` for trust visualization, and `Test-AdminAccess` to check local admin on remote hosts.
7. Use `Get-DomainUser -SPN` to list all SPN-set accounts (Kerberoasting targets).
8. Run Snaffler (`.\Snaffler.exe -d <DOMAIN> -s -v data`) to crawl readable shares for credentials, config files, SSH keys, and other sensitive data.
9. Run SharpHound (`.\SharpHound.exe -c All --zipfilename <NAME>`) to collect AD data, then upload the zip into BloodHound GUI.
10. In BloodHound, use the Analysis tab for pre-built queries: List all Kerberoastable Accounts, Find Computers with Unsupported Operating Systems, Find Computers where Domain Users are Local Admin, etc.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: How many Kerberoastable accounts in INLANEFREIGHT? | 13 | BloodHound → Analysis → List all Kerberoastable Accounts, count results |
| Q2: PowerView function to test admin access? | Test-AdminAccess | PowerView docs / section text |
| Q3: User in web config connection string? | sa | Snaffler output → web.config → connectionString uid= field |
| Q4: Password for the database user? | ILFREIGHTDB01! | Same Snaffler output → web.config → connectionString password= field |

## Key Takeaways
- The ActiveDirectory PowerShell module is stealthier than dropping tools — actions blend with normal admin activity.
- PowerView's `-Recurse` flag on `Get-DomainGroupMember` reveals nested group membership that standard tools miss (e.g., a user who is Domain Admin via a nested security group).
- `Test-AdminAccess` quickly identifies which hosts your current user has local admin on — critical for lateral movement planning.
- Snaffler automates the tedious process of hunting through shares for sensitive files — color-coded output (Red = high value) and regex matching for connection strings, keys, passwords.
- BloodHound pre-built queries instantly answer questions that would take hours manually (Kerberoastable accounts, shortest path to DA, unsupported OS, etc.).
- SharpView is the .NET alternative to PowerView — useful when PowerShell is restricted or monitored.
- Always document files transferred to/from domain hosts and clean up at engagement end.

## Gotchas
- BloodHound data must be re-collected if the environment changes — stale SharpHound data leads to missed paths or false positives.
- Snaffler produces massive output in large environments — always use `-o` to log to file and review afterward.
- The `backupagent` account in the Backup Operators group is a high-value target — Backup Operators can extract the AD database (ntds.dit) for offline credential extraction.
- Unsupported OS hosts shown in BloodHound may not be live — always validate before attacking or reporting.
- `admincount=1` on a user (like mmorgan) indicates they were once in a privileged group — even if removed, the AdminSDHolder ACL may still apply, making them a juicy target.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[14-credentialed-enum-linux]] | [[16-living-off-the-land]] →
<!-- AUTO-LINKS-END -->
