# Credential Hunting — Windows Privilege Escalation

## ID
530

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 21 — Credential Hunting

## Description
Covers searching the Windows file system for plaintext credentials in config files, Chrome dictionaries, unattend.xml, PowerShell history, and DPAPI-protected PowerShell credential objects.

## Tags
credential-hunting, findstr, dpapi, powershell-history, windows-privesc

## Commands
- findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml
- gc 'C:\Users\<USER>\AppData\Local\Google\Chrome\User Data\Default\Custom Dictionary.txt' | Select-String password
- gc (Get-PSReadLineOption).HistorySavePath
- foreach($user in ((ls C:\users).fullname)){cat "$user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt" -ErrorAction SilentlyContinue}
- $credential = Import-Clixml -Path '<PATH_TO_XML>'; $credential.GetNetworkCredential().password

## What This Section Covers
Application config files, browser dictionaries, unattended install files, and PowerShell history are common places where credentials are stored in cleartext or weakly protected form. This section teaches systematic file-system searching plus DPAPI-based credential decryption for PowerShell `Export-Clixml` files.

## Methodology

### 1 — Broad File-System Search with findstr
Start from `C:\Users` (or wherever makes sense) and cast a wide net:
```
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml
```
Flag breakdown: `/S` = recursive, `/I` = case-insensitive, `/M` = print only matching filenames. Adding `/C:` treats the search string as a literal.

Key locations to eyeball in the output:
- `C:\inetpub\wwwroot\web.config` — IIS creds
- `C:\Windows\Panther\unattend.xml` — auto-logon passwords (plaintext or base64)
- `Public\Documents\` and user `Documents\` folders — ad-hoc config files

### 2 — Chrome Dictionary Files
Users sometimes type passwords into browser fields that trigger spell-check. If they click "Add to dictionary," the password lands here:
```powershell
gc 'C:\Users\htb-student\AppData\Local\Google\Chrome\User Data\Default\Custom Dictionary.txt' | Select-String password
```

### 3 — Unattended Installation Files
`unattend.xml` files used during Windows imaging can contain auto-logon credentials in plaintext or base64. They should be auto-deleted after install, but sysadmins often leave copies around. Search for them:
```
dir /S /B C:\*unattend*.xml
```

### 4 — PowerShell History
PowerShell 5.0+ logs all commands to:
`C:\Users\<user>\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt`

Read current user's history:
```powershell
gc (Get-PSReadLineOption).HistorySavePath
```

Read ALL users' histories (re-run after getting admin — you'll see files you couldn't before):
```powershell
foreach($user in ((ls C:\users).fullname)){cat "$user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt" -ErrorAction SilentlyContinue}
```
Look for commands that pass creds inline: `wevtutil`, `net use`, `runas`, `cmdkey`, `Invoke-Command -Credential`.

### 5 — Decrypt DPAPI-Protected PowerShell Credentials
Scripts sometimes use `Get-Credential | Export-Clixml` to store encrypted creds in XML. If you have a shell as the same user who created the file:
```powershell
$credential = Import-Clixml -Path 'C:\scripts\pass.xml'
$credential.GetNetworkCredential().username
$credential.GetNetworkCredential().password
```
DPAPI ties decryption to the user + machine — running this as a different user gives nothing.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Search file system for password | Pr0xyadm1nPassw0rd! | `findstr` from `C:\Users` → `Public\Documents\settings.xml` contains proxy password |
| Q2: Decrypt pass.xml as bob, get flag | (flag on Desktop) | RDP as `bob:Str0ng3ncryptedP@ss!` → `Import-Clixml` on pass.xml → read flag.txt on Desktop |

## Key Takeaways
- `findstr /SIM` is your first move on any Windows box — fast, built-in, catches lazy config files.
- Always re-run PowerShell history and file searches after getting admin — you'll see other users' files.
- DPAPI-protected creds are only decryptable in the context of the user who created them.
- Chrome dictionary files are an overlooked goldmine — users unknowingly add passwords to their spellcheck dictionary.

## Gotchas
- `findstr` output is noisy — most hits are UEV templates and system XML. Focus on user-writable paths like Documents, Desktop, Public.
- `Set-ExecutionPolicy Bypass -Scope Process` may throw an error if a GPO overrides it, but the effective policy might already be Unrestricted — check with `Get-ExecutionPolicy`.
