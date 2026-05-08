## ID
532

## Module
Active Directory Enumeration & Attacks

## Kind
theory

## Title
Section 32 — Hardening Active Directory

## Description
Defensive countermeasures for common AD attack techniques covered in the module. Emphasises documentation, people/process/technology controls, the Protected Users group, and MITRE ATT&CK mappings for detection & mitigation.

## Tags
defense, hardening, mitre, protected-users, policy

## Key Concepts
- **Document & Audit** – Know every OU, GPO, FSMO role, trust, elevated user, and host.
- **People** – Strong passwords, no shared local admins, split-tier administration, Protected Users group for sensitive accounts.
- **Process** – Regular asset inventories, access control policies, decommissioning procedures, audit schedules.
- **Technology** – Tools like BloodHound, PingCastle, Grouper to find misconfigurations; disable NTLM, SMB signing, LDAP signing; remove unconstrained delegation; set `ms-DS-MachineAccountQuota` to 0; disable print spooler where possible.

## Protected Users Group
- Prevents credential caching (plaintext, Kerberos long-term keys), disallows NTLM & DES/RC4, limits TGT lifetime to 4 hours.
- Should be rolled out carefully to avoid lockouts.

## Protections by TTP (Table)
| TTP | MITRE Tag | Defence |
|-----|-----------|---------|
| External Recon | T1589 | Scrub public docs, job postings, metadata |
| Internal Recon | T1595 | Monitor network for scanning, block ICMP, use NIDS |
| Poisoning | T1557 | Enable SMB signing, strong encryption |
| Password Spraying | T1110.003 | Log events 4624/4648, lockout policy, MFA |
| Credentialed Enum | TA0006 | Detect anomalous CLI/RDP usage, network segmentation |
| LOTL (Living off Land) | N/A | Baseline traffic, AppLocker, restrict host applications |
| Kerberoasting | T1558.003 | Use AES encryption, strong passwords, gMSA accounts |

## Key Takeaways
- Defence begins with full visibility – document everything in AD.
- Combine people (training, least privilege), process (audit, cleanup), and technology (hardening settings) for a layered defence.
- The Protected Users group is a powerful built-in mitigation for credential theft on privileged accounts.
- Mitigations map directly to MITRE ATT&CK techniques; use the framework to prioritise controls.
- Even simple steps like disabling print spooler, setting `MachineAccountQuota=0`, and enabling SMB/LDAP signing block many common attack paths.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[31-cross-forest-trust-abuse-linux]] | [[36-beyond-this-module]] →
<!-- AUTO-LINKS-END -->
