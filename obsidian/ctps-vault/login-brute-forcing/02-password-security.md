# THEORY — Password Security Fundamentals

## ID
501

## Module
Password Attacks

## Kind
notes

## Title
Section 2 — Password Security Fundamentals

## Description
Explains what makes a password strong, common weaknesses, password policies, and the persistent danger of default credentials — and how these factors directly affect brute‑force attacks.

## Tags
password-security, theory, defense, best-practices, default-credentials

## TL;DR — What's Important
- **Length beats complexity every time:** A 12‑character random password is far stronger than an 8‑character mix of symbols. Each added character multiplies guess count exponentially.
- **Passphrases are the modern recommendation:** NIST now favors long, memorable phrases (e.g., `correct-horse-battery-staple`) over forced complexity like `P@ssw0rd!`.
- **Reusing passwords breaks all other protections:** If one service is breached, attackers will try that same password everywhere else (credential stuffing).
- **Default credentials are a goldmine for attackers:** Lists of `admin/admin`, `root/root`, `ubnt/ubnt` are built into every brute‑forcing tool. Changing the password but keeping `admin` as username still helps attackers.
- **Password policies have a dark side:** Overly strict policies (90‑day expiry, complex characters) push users to write passwords down or make predictable patterns like `Spring2024!` → `Summer2024!`.

## Concept Overview
Password security fundamentals determine how resistant a system is to brute‑force and other guessing attacks. Strong passwords have length, randomness, and uniqueness, while weak passwords are short, common, or reused. Organizations enforce password policies to mandate minimum standards, but poorly designed policies can backfire. One of the most overlooked vulnerabilities is default credentials shipped with hardware and software – attackers maintain extensive lists of these, making them an easy win during penetration tests. Understanding these principles helps pentesters predict which accounts are most vulnerable and helps defenders build effective authentication controls.

## Key Concepts

### Anatomy of a Strong Password (NIST SP 800‑63B)
- **Length** – Minimum 12 characters; longer is always better. NIST no longer recommends arbitrary complexity requirements (e.g., "must have uppercase, number, symbol").
- **All printable ASCII** (including spaces) is allowed – encourages passphrases.
- **No composition rules** – complexity rules often lead to predictable patterns (e.g., `Password1!`).
- **No periodic expiry** unless evidence of compromise. Forced rotation leads to weaker passwords.
- **Check against breach dictionaries** – block known common/leaked passwords.

### Common Password Weaknesses
| Weakness | Example | Why It’s Bad |
|----------|---------|---------------|
| Short length | `abc123` | Trivial to brute force (few million combos) |
| Dictionary word | `password`, `monkey` | Instant hit in wordlist attacks |
| Personal info | `John1985` | Easily scraped from social media |
| Reuse | same password for Gmail and corporate VPN | Credential stuffing across services |
| Predictable patterns | `qwerty`, `1qaz2wsx`, `p@ssw0rd` | Common substitutions are in rulesets |
| Keyboard walks | `1qaz@WSX` | Looks complex but is a simple pattern |

### Default Credentials – The Low‑Hanging Fruit

Default usernames and passwords are pre‑set by manufacturers. Attackers compile massive lists (e.g., SecLists, default‑password databases). Even if the password is changed, keeping the default username (e.g., `admin`) cuts the guessing work in half.

**Famous examples:**

| Device/Manufacturer | Default Username | Default Password |
|---------------------|------------------|------------------|
| Linksys Router | admin | admin |
| D‑Link Router | admin | admin |
| Netgear Router | admin | password |
| Cisco Router | cisco | cisco |
| Ubiquiti UniFi AP | ubnt | ubnt |
| Hikvision DVR | admin | 12345 |
| Axis IP Camera | root | pass |
| Canon Printer | admin | admin |
| Honeywell Thermostat | admin | 1234 |

**Why default credentials are a huge risk:**
- They are publicly documented.
- Many devices never get reconfigured.
- Automated scanning tools (e.g., `nmap` scripts, `hydra`, `medusa`) have built‑in default credential checks.
- Compromising one device (e.g., a printer) often leads to lateral movement into the corporate network.

### Password Policies – Balancing Security and Usability

Organizations use password policies to enforce minimum standards. Common settings:

- **Minimum password length** – typically 8–12 characters.
- **Complexity requirements** – e.g., 3 of 4 character classes.
- **Password expiration** – e.g., every 90 days.
- **Password history** – prevent reuse of last N passwords.

**Problems with over‑restrictive policies:**
- Users write passwords on sticky notes.
- Users increment numbers (`Spring2024!` → `Summer2024!`).
- Users choose weaker but compliant patterns (`P@ssw0rd1`).
- NIST now discourages arbitrary expiry without evidence of compromise.

## Why It Matters
For a penetration tester, password security fundamentals directly drive the attack strategy:
- **Weak / short passwords** → dictionary or simple brute force.
- **Default credentials** → immediate win; check `admin:admin` before anything else.
- **Reused passwords** → credential stuffing across internal apps or external services.
- **Strong policies** → may require offline hash cracking (if you can get hashes) or password spraying.

For defenders, these fundamentals are the blueprint for reducing the most common attack vector: compromised credentials. MFA is the ultimate answer, but strong password hygiene significantly raises the bar.

## Defender Perspective

### Detection
- **Failed login monitoring** – spikes in `admin` or `root` attempts suggest default credential scanning.
- **Password spray patterns** – one password across many accounts.
- **Credential stuffing detection** – unusual geolocations / user agents for a known valid account.

### Mitigations
- **Block default credentials** – force password change on first login.
- **Use breached password detection** (Azure AD Password Protection, HaveIBeenPwned API).
- **Enforce MFA** – makes password brute force irrelevant.
- **Adopt modern NIST guidelines** – longer passphrases, no periodic expiry, no arbitrary complexity.
- **Implement rate limiting and lockout** – but beware DoS risk.

### MITRE ATT&CK Mapping
- **T1110.001** – Password Guessing (online brute force)
- **T1110.003** – Password Spraying
- **T1110.004** – Credential Stuffing
- **T1078.001** – Default Accounts

## Key Takeaways
- A 12‑character passphrase (e.g., `coffee-morning-window`) is stronger than an 8‑character `P@ssw0rd!` because of length – and easier to remember.
- Attackers don't need to brute force if the password is `admin` or `password`. Always try default credentials first.
- Password reuse across accounts is the #1 reason credential stuffing works. Unique passwords per account compartmentalize breaches.
- NIST has moved away from complexity and periodic expiration – these cause more harm than good.
- In a pentest, enumerate password policy before attacking (e.g., via SMB null session or registration page) to avoid account lockouts.

## Gotchas
- **Do not assume a device is secure because the default password was changed** – if the default username `admin` remains, you still have a 50% head start.
- **Password expiration policies often backfire** – users rotate `Password1` → `Password2` → `Password3`, which is trivial to guess.
- **Complexity requirements alone don't prevent brute force** – `Password1!` is weak but meets most policies. Length is the real defense.
- **Some devices have hidden default credentials** – e.g., backdoor accounts documented only in service manuals. Always research the specific model.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|login-brute-forcing]]  
← [[01-intro]] | [[03-pin-cracking]] →
<!-- AUTO-LINKS-END -->
