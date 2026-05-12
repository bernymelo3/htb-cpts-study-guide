# NOTE — Spraying, Stuffing, and Defaults

## ID
510

## Module
Password Attacks

## Kind
methodology

## Title
Section 9 — Spraying, Stuffing, and Defaults

## Description
Three online attacks against many accounts at once: password spraying (one password, many users), credential stuffing (leaked user:pass pairs), and default-credential checks (vendor logins).

## Tags
password-spraying, credential-stuffing, default-credentials, kerbrute, netexec, creds-cheat-sheet

## Commands
- `netexec smb 10.100.38.0/24 -u <users.list> -p 'ChangeMe123!'`
- `hydra -C user_pass.list ssh://<ip>`
- `pip3 install defaultcreds-cheat-sheet`
- `creds search <vendor>`
- `kerbrute passwordspray -d <domain> --dc <dc-ip> users.txt 'Spring2024!'`

## TL;DR — What's Important
- **Spraying** ≠ **stuffing** ≠ **defaults**. Different attacks, different scenarios.
- All three are *online* attacks — be mindful of lockout policies.
- One password, many users (spray) is far less noisy per-account than brute-forcing a single user with many passwords.

## The Three Attacks

| Attack | What | Best For |
|--------|------|----------|
| **Password spray** | One password, many usernames | AD environments with weak common passwords (`Welcome1`, `Password123`, `<Season><Year>!`) |
| **Credential stuffing** | List of `user:pass` pairs from a breach | Targets where password reuse across services is suspected |
| **Default credentials** | Vendor defaults (`admin:admin`, etc.) | Routers, switches, IoT, dev/staging boxes, databases |

## Password Spraying

### SMB (AD)
```bash
netexec smb 10.100.38.0/24 -u users.list -p 'ChangeMe123!'
```

### Kerberos (pre-auth, no SMB needed)
```bash
kerbrute passwordspray -d inlanefreight.htb --dc 10.129.0.10 users.txt 'Spring2024!'
```
Kerbrute is quieter than SMB spray because it skips the SMB layer entirely.

### Lockout-Safe Pattern
1. Pull the account lockout threshold from the domain policy (`net accounts /domain` or `Get-ADDefaultDomainPasswordPolicy`).
2. Spray *one* password per `<lockout_observation_window>` to never lock an account.

## Credential Stuffing
If you have leaked `user:pass` pairs from one service, try them on others:
```bash
hydra -C user_pass.list ssh://10.100.38.23
```
Format of `user_pass.list`:
```
user1:pass1
user2:pass2
```

## Default Credentials

### `defaultcreds-cheat-sheet`
```bash
pip3 install defaultcreds-cheat-sheet
creds search linksys
creds search mysql
```
Output is a table of `Product | username | password` — pipe into a list for spraying.

### Common Router Defaults

| Vendor | Default IP | User | Pass |
|--------|------------|------|------|
| Belkin | 192.168.2.1 | admin | admin |
| D-Link | 192.168.0.1 | admin | Admin |
| Linksys | 192.168.1.1 | admin | Admin |
| Netgear | 192.168.0.1 | admin | password |
| 3Com | 192.168.1.1 | admin | Admin |

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — Retrieve MySQL creds from the target (after SSH login as `sam`) | **(hidden — username:password format)** | `creds search mysql` to discover defaults; try `mysql -h localhost -usuperdba -p` with password `admin` |

## Key Takeaways
- Spraying with `<Season><Year>!` (e.g. `Spring2024!`, `Summer2024!`) hits 5–15% of large AD environments. It's embarrassingly effective.
- Default credentials are *not* limited to home routers — staging databases, internal monitoring tools (Nagios, Grafana), and printers are frequent culprits.
- A successful spray against AD = check what shares / endpoints the cracked account has access to *before* escalating noise.

## Gotchas
- Lockout policies on AD can lock domain accounts in seconds. Always check threshold + observation window first.
- Some vendor defaults require initial-setup endpoints that get disabled after first config. Test against fresh installs.
- Credential stuffing presumes password reuse. If the target enforces unique passwords-per-service, this fails fast.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[08-network-services]] | [[10-windows-authentication-process]] →
<!-- AUTO-LINKS-END -->
