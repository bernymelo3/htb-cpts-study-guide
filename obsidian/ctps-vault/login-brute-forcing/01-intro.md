# THEORY — Login Brute Forcing

## ID
500

## Module
Password Attacks

## Kind
notes

## Title
Section 1 — Login Brute Forcing

## Description
A comprehensive introduction to brute forcing: what it is, how it works, the main attack variations, and its strategic role in penetration testing.

## Tags
brute-forcing, password-attacks, theory, methodology, authentication

## TL;DR — What's Important
- **Brute forcing is trial-and-error:** Systematically tries every possible combination until the correct credential is found. Success depends on password complexity, attacker compute power, and target defenses.
- **Not all brute force is the same:** Dictionary attacks, password spraying, credential stuffing, and rainbow tables each have different trade-offs. Choose based on available intel and lockout policies.
- **Password spraying beats lockouts:** Trying one weak password against many usernames avoids account lockouts and stays under detection radar — often more effective than attacking one account.
- **Simple brute force is last resort:** Trying all combinations of a 8‑character alphanumeric password is impractical. Real attacks use dictionaries, rules, or leaked credential lists.
- **Penetration testers use brute force after other paths fail:** It’s not the first tool, but it’s essential for testing password policy strength and finding reused credentials.

## Concept Overview
Brute forcing is a trial‑and‑error method to guess passwords, PINs, or encryption keys by systematically trying many possibilities. In web applications and network services, attackers submit login requests with generated credentials until one works. The attack’s success depends on password complexity (entropy), available computational power, and any defensive mechanisms like rate limiting or account lockouts. While simple brute force (exhaustive search) is often impractical for modern passwords, variants such as dictionary attacks, password spraying, and credential stuffing are highly effective in real‑world assessments.

## Key Concepts

### How Brute Forcing Works (Flow)
1. **Start** – Attacker initiates attack with parameters (character set, length, wordlist).
2. **Generate candidate** – Software produces next password guess.
3. **Apply** – Candidate is submitted to target (login form, API, SMB, SSH, etc.).
4. **Check success** – System returns success/failure. If success → access granted. If failure → loop back to step 2.
5. **Access granted** – Valid credentials found; attacker gains entry.

### Types of Brute Forcing (Comparison Table)

| Method | Description | Example | Best Used When... |
|--------|-------------|---------|--------------------|
| Simple Brute Force | Tries every combination from a character set (e.g., a-z, A-Z, 0-9). | Trying all 4‑6 character lowercase passwords. | No prior info & massive compute (e.g., short password length). |
| Dictionary Attack | Uses a pre‑compiled wordlist of common passwords. | `rockyou.txt` against a login form. | Target likely uses weak/common passwords. |
| Hybrid Attack | Dictionary + mutations (append/prepend numbers/symbols). | `password` → `password123`, `Password!` | Users slightly modify common passwords. |
| Credential Stuffing | Reuses leaked username/password pairs from one breach on other services. | Using a breach dump to log into unrelated sites. | Large leaked credential sets available; users reuse passwords. |
| Password Spraying | One (or a few) weak passwords tried against many usernames. | `Password123` against all `@company.com` accounts. | Account lockout policies exist; avoid per‑account rate limits. |
| Rainbow Table Attack | Pre‑computed hash chains to reverse password hashes quickly. | Look up an NTLM hash in a pre‑computed table. | You have many password hashes and storage for tables. |
| Reverse Brute Force | One password tried against many usernames (subset of spraying). | Leaked password `Summer2024!` against many accounts. | A specific password is suspected to be reused widely. |
| Distributed Brute Force | Split workload across many machines (cluster, botnet, GPU farm). | Hashcat in distributed mode across 10 GPUs. | Password is long/complex and single machine is too slow. |

## Why It Matters
Brute forcing is not just a “noisy, last‑resort” attack — it’s a practical way to test the real‑world strength of an organization’s authentication controls. In penetration tests, brute forcing is used to:
- Validate password policy enforcement (e.g., are users choosing `Password123`?).
- Uncover credential reuse across internal services (e.g., same password for VPN and SharePoint).
- Gain initial foothold when no other vulnerability exists.
- Demonstrate the impact of missing account lockout / MFA.

Without brute force testing, a pentest might miss the most common entry vector: weak passwords.

## Defender Perspective

### Detection Signs
- **High volume of failed logins** from a single IP (simple brute force).
- **Many unique usernames with the same failed password** (password spraying).
- **Unusual login timestamps** (e.g., 3 AM bursts of 100 attempts/minute).
- **Login patterns from unexpected geolocations** or user agents.

### Mitigations
- **Account lockout** after N failed attempts (but beware – spraying works around this).
- **Rate limiting** per IP / per username.
- **CAPTCHA** after 3–5 failures.
- **MFA (Multi‑Factor Authentication)** – the strongest defense; brute force on password alone is useless if MFA is enforced.
- **Block known breached passwords** (e.g., Azure AD Password Protection).
- **Long, complex password policies** (but user friction trade‑off).

### MITRE ATT&CK Mapping
- **T1110.001** – Brute Force: Password Guessing  
- **T1110.002** – Brute Force: Password Cracking (offline)  
- **T1110.003** – Brute Force: Password Spraying  
- **T1110.004** – Brute Force: Credential Stuffing

## Key Takeaways
- Simple brute force is rarely practical for online services due to lockouts and speed limits; dictionary and spraying are far more effective.
- Password spraying is stealthier than traditional brute force because it avoids per‑account failure thresholds.
- Credential stuffing is not technically “brute forcing” (it uses known credentials) but is often grouped here because tools overlap.
- Rainbow tables are obsolete for modern salted hashes but still relevant for legacy systems (LM hashes, unsalted MD5).
- In penetration testing, always check account lockout policy first – otherwise you might lock out real users.

## Gotchas
- **Do not use online brute force without permission** – it can lock out accounts and trigger incident response.
- **Password spraying can still trigger alerts** if the same password is tried against every account in quick succession (use random delays / low‑and‑slow).
- **Simple brute force on a modern 8‑character mixed‑case + digit + symbol password** would take centuries on a single CPU. Always prefer wordlists + rules.
- **If the target uses MFA, brute forcing the password alone is wasted time** – unless you can bypass MFA (session hijacking, MFA fatigue, etc.).

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|login-brute-forcing]]  
[[02-password-security]] →
<!-- AUTO-LINKS-END -->
