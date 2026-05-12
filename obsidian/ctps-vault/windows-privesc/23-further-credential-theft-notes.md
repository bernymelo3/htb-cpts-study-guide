# Further Credential Theft — Windows Privilege Escalation

## ID
532

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 23 — Further Credential Theft

## Description
Covers cmdkey/runas /savecred, SharpChrome, KeePass hash extraction, LaZagne, SessionGopher, Windows AutoLogon registry creds, PuTTY proxy passwords in the registry, and Wi-Fi PSK extraction.

## Tags
lazagne, sharpchrome, sessiongopher, keepass, registry-creds, windows-privesc

## Commands
- cmdkey /list
- runas /savecred /user:<DOMAIN>\<USER> "<COMMAND>"
- .\SharpChrome.exe logins /unprotect
- python2.7 keepass2john.py <KDBX_FILE>
- hashcat -m 13400 <HASH_FILE> <WORDLIST>
- .\lazagne.exe all
- Import-Module .\SessionGopher.ps1; Invoke-SessionGopher -Target <HOSTNAME>
- reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
- reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions
- netsh wlan show profile <SSID> key=clear

## What This Section Covers
A grab-bag of post-exploitation credential extraction techniques beyond basic file searching. Covers saved Windows credentials (cmdkey), browser passwords (SharpChrome), password manager databases (KeePass), multi-app harvesters (LaZagne), session credential extractors (SessionGopher), cleartext registry passwords (AutoLogon, PuTTY proxy), and Wi-Fi pre-shared keys.

## Methodology

### 1 — Cmdkey Saved Credentials
List any credentials Windows has cached for the current user:
```cmd
cmdkey /list
```
If you see entries (especially `TERMSRV/*` for RDP), you can reuse them with `runas /savecred` without knowing the password:
```powershell
runas /savecred /user:inlanefreight\bob "cmd.exe /c whoami > C:\temp\whoami.txt"
```
This is a direct privilege escalation path if the saved creds belong to a higher-privilege account.

### 2 — Browser Credentials (SharpChrome)
Decrypts Chrome saved logins using the DPAPI AES state key:
```powershell
.\SharpChrome.exe logins /unprotect
```
Output shows URL, username, and decrypted password for each saved login. Must run as the user who owns the Chrome profile. Blue team detection: generates 4688 (process creation) and 16385 (DPAPI activity) events.

### 3 — Password Manager Databases (KeePass)
If you find a `.kdbx` file on the target or a share, exfil it and crack offline:
```bash
python2.7 keepass2john.py ILFREIGHT_Help_Desk.kdbx
```
Then crack with Hashcat (mode 13400):
```bash
hashcat -m 13400 keepass_hash /usr/share/wordlists/rockyou.txt
```
A cracked KeePass DB used by IT staff can give you keys to the kingdom — servers, network devices, databases.

### 4 — LaZagne (Multi-App Credential Harvester)
Dumps saved passwords from browsers, chat clients, databases, email, sysadmin tools, Wi-Fi, credential manager, DPAPI, LSA secrets, and more:
```powershell
.\lazagne.exe all
```
Can also target specific modules: `.\lazagne.exe browsers`, `.\lazagne.exe databases`, `.\lazagne.exe sysadmin`, etc. View all modules with `.\lazagne.exe -h`.

### 5 — SessionGopher (Session Credential Extractor)
Extracts saved session info from PuTTY, WinSCP, SuperPuTTY, FileZilla, and RDP by searching `HKEY_USERS` registry hives:
```powershell
Import-Module .\SessionGopher.ps1
Invoke-SessionGopher -Target WINLPE-SRV01
```
Needs local admin to enumerate all users' hives, but always worth running as current user. Also searches drives for `.ppk`, `.rdp`, and `.sdtid` files.

### 6 — Windows AutoLogon (Registry)
If AutoLogon is configured, the username and password are stored in cleartext in the registry:
```cmd
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```
Look for `AutoAdminLogon = 1`, `DefaultUserName`, and `DefaultPassword`. Any standard user can read this key.

### 7 — PuTTY Proxy Credentials (Registry)
PuTTY sessions using a proxy store the proxy username and password in cleartext in the registry:
```powershell
reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions
```
Then query each session:
```powershell
reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions\<SESSION_NAME>
```
Look for `ProxyUsername` and `ProxyPassword`. Tied to the user who saved the session — needs their `HKCU` or admin access to `HKU`.

### 8 — Wi-Fi Passwords
List saved wireless profiles:
```cmd
netsh wlan show profile
```
Extract the PSK for a specific network:
```cmd
netsh wlan show profile ilfreight_corp key=clear
```
The password appears under `Key Content`. Requires local admin on a machine with a wireless card.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: sa password for SQL01.inlanefreight.local | S3cret_db_p@ssw0rd! | RDP as `jordan:HTB_@cademy_j0rdan!` → `lazagne.exe all` → DbVisualizer saved creds |
| Q2: User with saved RDP creds for WEB01 | amanda | RDP as `htb-student` → Open Remote Desktop Connection → type `WEB01` → username auto-populates |
| Q3: root password for vc.inlanefreight.local/ui | ILVCadm1n1qazZAQ! | `SharpChrome.exe logins /unprotect` → Chrome saved login for vc01 |
| Q4: Password for ftp.ilfreight.local | Ftpuser! | `SessionGopher` → WinSCP saved session for `root@ftp.ilfreight.local` |

## Key Takeaways
- `cmdkey /list` + `runas /savecred` is a direct privesc path that doesn't require cracking anything — if creds are cached, you just use them.
- LaZagne is the broadest single tool for credential extraction — covers dozens of apps in one sweep.
- Registry is full of cleartext passwords: AutoLogon, PuTTY proxy sessions, and more. Always check `Winlogon` and `SimonTatham\PuTTY`.
- KeePass databases on shares or desktops are high-value targets — IT staff often use them, and weak master passwords crack fast.
- Wi-Fi PSKs can give access to separate network segments — rare but valuable when it hits.

## Gotchas
- `runas /savecred` only works if the credentials are already cached via `cmdkey` — it won't prompt for a password, it just fails silently if nothing is stored.
- SharpChrome must run as the user who owns the Chrome profile — DPAPI binds decryption to user context.
- LaZagne needs to run as the target user for DPAPI-backed stores. Running as a different user only gets you non-DPAPI creds (e.g., config files).
- PuTTY session registry keys are under `HKCU`, so you only see the current user's sessions unless you have admin and can enumerate `HKU\<SID>`.
- The Q2 answer (amanda) is found via the GUI auto-populate, not via command-line tools — don't overthink it.
