# NOTE — Password Managers

## ID
531

## Module
Password Attacks

## Kind
reference

## Title
Section 25 — Password Managers

## Description
Cloud vs local password managers, zero-knowledge encryption design, the master-key/master-password split, and passwordless authentication alternatives (FIDO2, TOTP, MFA).

## Tags
password-manager, bitwarden, 1password, keepass, fido2, mfa, totp, passwordless, defense

## TL;DR — What's Important
- Cloud managers use **zero-knowledge encryption** — the provider can't read your vault even with a server breach (in theory).
- Local managers shift the storage + backup risk onto you — different threat model, not strictly safer.
- A weak master password defeats both. Pick a long passphrase + enable MFA on the manager itself.

## Why Use One At All
Average user has ~100 passwords (NordPass 2023). Without a manager, they reuse 3–5 passwords across all of them. Password reuse is the #1 enabler of credential stuffing.

## How Cloud Managers Work (Zero-Knowledge)
The master password never leaves the client. Three derived values:
1. **Master Key** — derived from master password via PBKDF2/Argon2 (high iteration count)
2. **Master Password Hash** — different derivation from the master key, used to *authenticate* to the cloud server. Server stores this hash; can't reverse it.
3. **Decryption Key** — derived from master key, used to AES-decrypt vault entries client-side.

Server stores: encrypted vault + master-password hash. Server *cannot* decrypt the vault.

```
Master Password ──PBKDF2──> Master Key ──┬──> Master Password Hash → Cloud auth
                                          └──> Decryption Key       → AES decrypt vault
```

## Cloud vs Local — Trade-offs

| Aspect | Cloud (Bitwarden, 1Password, Dashlane) | Local (KeePass, Password Safe) |
|--------|---------------------------------------|--------------------------------|
| Sync across devices | Automatic | Manual (sync via Dropbox/Drive/your storage) |
| Server breach risk | Encrypted vault leaks; hard to crack with strong master pass | None (no server) |
| Master pass loss | Permanent vault loss (provider can't help) | Permanent vault loss |
| Convenience | High | Lower |
| Trust model | Trust client + provider's claim of zero-knowledge | Trust yourself + your backups |

## Notable Cloud Managers
- **Bitwarden** — open-source, self-host option
- **1Password** — adds "secret key" alongside master pass (two-factor by design)
- **Dashlane**, **NordPass**, **RoboForm**, **Keeper**, **LastPass** (history of breaches)

## Notable Local Managers
- **KeePass** / **KeePassXC** — open-source, `.kdbx` format
- **Password Safe** (`.psafe3`) — by Bruce Schneier's team
- **KWalletManager** — KDE-integrated

## How Attackers Approach Them
- **`.kdbx` files** — crack offline with `keepass2john` → hashcat `-m 13400`. Effort scales with KDF iteration count.
- **`.psafe3` files** — hashcat `-m 5200`.
- **Browser plugin auto-fill** — DOM injection, clipboard scraping. Use copy-and-paste with auto-clear timeout, not autofill.
- **Memory dump of unlocked vault** — Mimikatz `memory::keepass` reads the master key from running KeePass process.
- **Phishing the master password** — fake browser unlock prompt.

## Alternatives — Going Passwordless

| Mechanism | What |
|-----------|------|
| **MFA** | Something you know + have/are. Push (Duo), TOTP (Google Auth, Authy), SMS (weak). |
| **TOTP / HOTP** | RFC 6238 — 6-digit codes rotating every 30s |
| **FIDO2 / WebAuthn** | Hardware-token (YubiKey) or platform authenticator (Windows Hello, Touch ID). Phishing-resistant. |
| **Smart cards / CAC** | Cert-based; common in government + healthcare |
| **Push notification** | Tap-to-approve on a registered device |

FIDO2 is the gold standard — phishing-resistant by design (origin binding). Major identity providers (Azure AD, Okta, Auth0, Google) all support passwordless FIDO2.

## Microsoft / Vendor Passwordless Pushes
- [Microsoft Passwordless](https://www.microsoft.com/security/business/identity-access/azure-active-directory-passwordless-authentication)
- [Auth0 Passwordless](https://auth0.com/docs/authenticate/passwordless)
- [Okta Passwordless](https://www.okta.com/passwordless-authentication/)
- [Ping Identity](https://www.pingidentity.com/)

## Recommendations
| Threat Model | Recommendation |
|--------------|----------------|
| Personal — convenience priority | Cloud manager (Bitwarden free or 1Password) + FIDO2 key for the manager itself |
| Personal — paranoid | KeePassXC on a tracked-backup setup |
| Enterprise SSO | Azure AD / Okta + FIDO2 keys, no shared admin accounts |
| Pentester | KeePassXC + USB token for engagement creds; never reuse personal-manager creds in work flows |

## Key Takeaways
- Zero-knowledge encryption means the cloud provider holds an *encrypted blob* — vault security still rides entirely on the master password's strength + MFA on the manager.
- Local-vs-cloud is a *threat model* choice, not a "safer/less safe" axis. Both can lose the vault permanently if the master pass is forgotten.
- FIDO2 hardware tokens are phishing-resistant in a way that passwords + TOTP are not — they verify the origin URL.
- For pentesters: when you find a `.kdbx` file, crack it offline. When you find an *unlocked* KeePass running, dump its memory.

## Gotchas
- Cloud managers have been breached (LastPass 2022, OneLogin earlier). The encrypted vaults were extracted; cracking weak master passwords yielded plaintext. Strong master passphrase is the only real defense.
- KeePass's "Secure Desktop" prevents keyloggers from capturing the master password but only when the option is enabled — off by default.
- Some passwordless implementations still fall back to passwords on enrollment / recovery — read the docs before assuming a path is truly password-free.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[24-password-policies]] | [[26-skills-assessment]] →
<!-- AUTO-LINKS-END -->
