# Other Files — Windows Privilege Escalation

## ID
531

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 22 — Other Files

## Description
Covers manual file-system and share-drive credential hunting using findstr, dir, PowerShell, Sticky Notes SQLite extraction, and a checklist of high-value Windows files to always check.

## Tags
credential-hunting, findstr, sticky-notes, pssqlite, file-search, windows-privesc

## Commands
- cd C:\Users\<USER>\Documents & findstr /SI /M "password" *.xml *.ini *.txt
- findstr /si password *.xml *.ini *.txt *.config
- findstr /spin "password" *.*
- select-string -Path C:\Users\<USER>\Documents\*.txt -Pattern password
- dir /S /B *pass*.txt == *pass*.xml == *pass*.ini == *cred* == *vnc* == *.config*
- where /R C:\ *.config
- Get-ChildItem C:\ -Recurse -Include *.rdp, *.config, *.vnc, *.cred -ErrorAction Ignore
- Invoke-SqliteQuery -Database <DB_PATH> -Query "SELECT Text FROM Note" | ft -wrap
- strings plum.sqlite-wal

## What This Section Covers
Beyond the structured searches from the previous section, this covers deeper manual file-system hunting with multiple `findstr` variations, PowerShell equivalents, Sticky Notes database extraction via PSSQLite or `strings`, and a reference list of high-value Windows files that may contain credentials or useful config data.

## Methodology

### 1 — Manual File Content Searches (Multiple Approaches)

**CMD — filenames only:**
```cmd
cd C:\Users\htb-student\Documents & findstr /SI /M "password" *.xml *.ini *.txt
```

**CMD — show matching lines:**
```cmd
findstr /si password *.xml *.ini *.txt *.config
```

**CMD — show filename, line number, and match:**
```cmd
findstr /spin "password" *.*
```

**PowerShell equivalent:**
```powershell
select-string -Path C:\Users\htb-student\Documents\*.txt -Pattern password
```

### 2 — Search by Filename / Extension

**CMD — hunt for password-related filenames:**
```cmd
dir /S /B *pass*.txt == *pass*.xml == *pass*.ini == *cred* == *vnc* == *.config*
```

**CMD — find all .config files on C:\:**
```cmd
where /R C:\ *.config
```

**PowerShell — search for high-value extensions:**
```powershell
Get-ChildItem C:\ -Recurse -Include *.rdp, *.config, *.vnc, *.cred -ErrorAction Ignore
```

### 3 — Sticky Notes Database Extraction
Sticky Notes stores data in a SQLite DB at:
`C:\Users\<user>\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite`

**Option A — PSSQLite (on target):**
```powershell
cd C:\Tools\PSSQLite\
Set-ExecutionPolicy Bypass -Scope Process
Import-Module .\PSSQLite.psd1
$db = 'C:\Users\htb-student\AppData\Local\Packages\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite'
Invoke-SqliteQuery -Database $db -Query "SELECT Text FROM Note" | ft -wrap
```

**Option B — strings (on attack box after exfil):**
```bash
strings plum.sqlite-wal
```
The `-wal` (write-ahead log) file often contains the actual note text even if the main `.sqlite` file looks empty.

**Option C — DB Browser for SQLite (GUI):**
Copy all three `plum.sqlite*` files to your attack box, open in DB Browser, run `SELECT Text FROM Note;`.

### 4 — High-Value File Checklist
Always check these paths when hunting for creds or useful config data:

| Path | What's There |
|------|-------------|
| `%WINDIR%\repair\sam` / `system` / `security` | Backup SAM/SYSTEM hives |
| `%WINDIR%\debug\NetSetup.log` | Domain join info |
| `%WINDIR%\iis6.log` | Legacy IIS logs |
| `%WINDIR%\system32\config\*.sav` | Registry hive backups |
| `%WINDIR%\System32\drivers\etc\hosts` | Host entries (internal hostnames) |
| `%SYSTEMDRIVE%\pagefile.sys` | Virtual memory — can contain plaintext creds |
| `%USERPROFILE%\ntuser.dat` | User registry hive |
| `%WINDIR%\system32\CCM\logs\*.log` | SCCM logs |
| `C:\ProgramData\Configs\*` | App configs |
| `C:\Program Files\Windows PowerShell\*` | Custom PS modules/scripts |

### 5 — Network Share Hunting (AD Environments)
In AD, tools like **Snaffler** crawl network shares for high-value file types: `.kdbx` (KeePass), `.vmdk`/`.vhdx` (virtual disks — mount and extract hashes), `.ppk` (PuTTY private keys), `.rdp` files with saved creds, Excel/Word docs with passwords. User home folders mapped to shares are especially juicy — people store personal files thinking they're private.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Cleartext password for bob_adm | 1qazXSW@3edc! | PSSQLite → Sticky Notes `plum.sqlite` → `SELECT Text FROM Note` → `bob_adm:1qazXSW@3edc!` |

## Key Takeaways
- Sticky Notes is an underrated gold mine — people treat it like a password manager.
- `findstr` flags matter: `/M` for filenames-only (triage), `/spin` for full context (deep dive).
- The `-wal` file often has the real data — don't just grab `plum.sqlite`, grab all three files.
- Automated enumeration scripts don't catch everything — knowing the manual commands lets you customize searches on the fly.
- In AD environments, user share drives are some of the richest credential sources; Snaffler automates the crawl.

## Gotchas
- `dir /S /B` with `==` syntax is CMD-only — doesn't work in PowerShell.
- `Get-ChildItem -Recurse` on C:\ is slow and noisy; add `-ErrorAction Ignore` or you'll drown in access-denied errors.
- `Set-ExecutionPolicy Bypass` may fail with a GPO override error but the effective policy might already allow script execution — just proceed.
- `plum.sqlite` may appear empty if the user recently created notes — check `plum.sqlite-wal` which contains uncommitted writes.
