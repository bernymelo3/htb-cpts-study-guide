# NOTE — Password Policies

## ID
530

## Module
Password Attacks

## Kind
reference

## Title
Section 24 — Password Policies

## Description
What a password policy is, what to enforce (and what *not* to enforce — looking at you, 90-day expiry), industry standards (NIST 800-63B, CIS, PCI DSS), and what blacklist terms to add.

## Tags
password-policy, nist, cis, pci-dss, blacklist, defense, blue-team

## TL;DR — What's Important
- A *policy document* without *technical enforcement* is theater.
- Length > complexity. NIST 800-63B explicitly recommends against mandatory rotation + character-class requirements.
- Blacklist company name, seasons, months, common patterns — most weak passwords are predictable.

## What a Password Policy Defines
- Minimum length (often 12+ in modern guidance)
- Complexity (legacy — character classes required)
- Maximum age / rotation (legacy — modern NIST says don't)
- Reuse history
- Lockout threshold + duration
- Storage requirements (algorithm, salt, peppering)

## Reference Standards

| Standard | Notable Position |
|----------|------------------|
| **[NIST SP 800-63B](https://pages.nist.gov/800-63-3/sp800-63b.html)** | Min 8 chars. No mandatory rotation unless compromise detected. Allow all printable characters incl. spaces. Check against breach lists. |
| **CIS Password Policy Guide** | Min 14 chars; lockout after 5 attempts in 15 min; no reuse of last 24. |
| **PCI DSS** | Min 12 chars; rotate every 90 days; complexity required; lockout after 10. |

NIST and PCI are at opposite ends — NIST modernized 2017+; PCI still requires periodic rotation. Pick the one your industry mandates, layer additional controls on top.

## Why Mandatory Rotation Backfires
| Original Password | After 60 days | After 120 days |
|-------------------|---------------|----------------|
| `Inlanefreight01!` | `Inlanefreight02!` | `Inlanefreight03!` |
| `Summer2024!` | `Autumn2024!` | `Winter2024!` |

Predictable mutation — easily covered by a rule file. NIST 800-63B §5.1.1.2 explicitly says: "Verifiers SHOULD NOT require memorized secrets to be changed arbitrarily (e.g., periodically)." Only force a change on suspected compromise.

## Recommended Blacklist Entries
Block any password containing:
- Company name + common variations (`Inlanefreight`, `ILF`, `IL-Freight`)
- City/region (`SanFrancisco`, `SF`, `Bay`)
- Industry slang
- `password`, `qwerty`, `123456`, etc. (top-1000 list)
- Seasons + months + days
- `welcome`, `letmein`, `admin`

[HaveIBeenPwned Passwords](https://haveibeenpwned.com/Passwords) exposes a free API of breached passwords — integrate at registration/change time.

## Enforcement Mechanisms

| Environment | Tool |
|-------------|------|
| **Active Directory** | Default Domain Policy (GPO) — Password + Account Lockout |
| **AD (fine-grained)** | Password Settings Objects (PSOs) for per-group control |
| **Linux** | PAM modules: `pam_cracklib`, `pam_pwquality`, `pam_passwdqc`, `pam_unix` with `remember=N` |
| **Linux (centralized)** | FreeIPA / SSSD with central policy |
| **Web apps** | Backend validation + breach-list check (zxcvbn, HIBP API) |
| **All** | Hardware-token MFA — biggest single risk reducer |

## Creating Strong, Memorable Passwords
- **Passphrases** beat random strings on usability without sacrificing security: `correct-horse-battery-staple` (~44 bits if word list is known).
- **Diceware** generates passphrases from large word lists with measurable entropy.
- **Tools**: [1Password Password Generator](https://1password.com/password-generator), [Bitwarden generator](https://bitwarden.com/password-generator/), `pwgen -s 16 1`.

## Quick Strength Test
- [passwordmonster.com](https://www.passwordmonster.com/) — entropy estimate
- `zxcvbn` library — pattern-aware strength estimator (used in production)

## Key Takeaways
- The policy document and the technical control must agree. Documented "12 chars minimum" with AD configured for 7 = the AD config wins.
- Length is cheap; rotation is expensive (in user attention and predictability). Modern guidance: long, never rotate (unless breached), MFA on top.
- Blacklist the company name and obvious context — it's the single highest-yield blue-team control after MFA.
- Defenders should periodically dump NTDS and crack their own users with rockyou+rules to find weak passwords before attackers do.

## Gotchas
- Some legacy compliance regimes (PCI, HIPAA) still mandate rotation. Document the conflict if you push against it.
- "Strong password" UX must accept long inputs — many apps still cap at 16 or 20 chars (limiting passphrases).
- AD's Default Domain Policy applies to all domain users uniformly. For per-group differences, you need PSOs (a fine-grained password policy).

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[23-pass-the-certificate]] | [[25-password-managers]] →
<!-- AUTO-LINKS-END -->
