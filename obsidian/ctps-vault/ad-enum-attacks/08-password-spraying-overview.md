# THEORY — Password Spraying Overview (AD Enumeration & Attacks)

## ID
800  <!-- Confirm ID in the Active Directory Enumeration & Attacks module range -->

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 8 — Password Spraying Overview

## Description
Understand password spraying attacks: how they work, when to use them, and how to build effective target lists from OSINT and username enumeration to gain that crucial initial foothold in AD.

## Tags
password-spraying, active-directory, kerbrute, enumeration, theory, osint

## TL;DR — What’s Important
- **One password, many users:** spray a single common password (e.g. `Welcome1`, `Passw0rd`, `Winter2022`) against a long list of valid usernames to avoid account lockouts.
- **Username list is everything:** combine OSINT (LinkedIn, document metadata) with tools like Kerbrute to compile a high-quality target list; even a low-privilege hit opens the door for BloodHound analysis.
- **Respect the lockout policy:** if you don’t know the policy, leave hours between sprays; if you’re internal, enumerate the policy first (net accounts, crackmapexec, etc.).
- **Predictable username formats backfire:** the second scenario shows how a published PDF with an internal username format (F9L8) allowed enumeration of every domain account via a short Bash script.
- **Password spraying is not brute force:** it’s a stealthier, low-and-slow approach that trades many passwords for many usernames; still, monitor lockouts and have a plan B.

## Concept Overview
Password spraying is a low‑and‑slow authentication attack where one common password is tested against a large number of user accounts, instead of trying many passwords for one user. It’s especially effective in Active Directory environments where lockout policies prevent traditional brute force. The attack can be used both externally (e.g., OWA, VPN, RDP) and internally (SMB, Kerberos) to gain an initial foothold or move laterally. Because it sends few login attempts per user and introduces delays, it often flies under the radar of account lockout thresholds and alerting systems.

## Key Concepts

### Brute Force vs. Password Spraying
| Brute Force | Password Spraying |
|-------------|-------------------|
| Many passwords → one user | One password → many users |
| High lockout risk | Low lockout risk (with delays) |
| Quick, noisy | Slow, stealthy |
| Better for targeted accounts | Better for wide initial access |

### Building a Target User List
- **Active Directory domain username enumeration:** tools like Kerbrute can validate usernames against the DC without logging in.
- **OSINT:** scrape LinkedIn, company websites, public documents.
- **Document metadata:** authors, creators, and last‑modified‑by fields can reveal internal username formats.
- **Predictable naming schemes:** e.g., `first.last`, `F9L8` patterns can be generated with scripts.

```bash
#!/bin/bash
# Generate all possible 4-character usernames from A-Z,0-9 (base36)
for x in {{A..Z},{0..9}}{{A..Z},{0..9}}{{A..Z},{0..9}}{{A..Z},{0..9}}
    do echo $x;
done

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[07-llmnr-nbtns-poisoning-windows]] | [[09-enumerating-password-policies]] →
<!-- AUTO-LINKS-END -->
