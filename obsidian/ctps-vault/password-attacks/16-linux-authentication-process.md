# NOTE — Linux Authentication Process

## ID
522

## Module
Password Attacks

## Kind
theory

## Title
Section 16 — Linux Authentication Process

## Description
PAM-based auth on Linux: `/etc/passwd`, `/etc/shadow`, hash format (`$id$salt$hash`), opasswd history, and the `unshadow` workflow for offline cracking.

## Tags
linux, pam, passwd, shadow, unshadow, sha512crypt, yescrypt, opasswd, hashcat-1800, jtr

## Commands
- `cat /etc/passwd`
- `sudo cat /etc/shadow`
- `unshadow /etc/passwd /etc/shadow > unshadowed.hashes`
- `hashcat -m 1800 -a 0 unshadowed.hashes rockyou.txt`
- `john unshadowed.hashes`
- `sudo cat /etc/security/opasswd`

## TL;DR — What's Important
- `/etc/passwd` (world-readable) lists accounts; `/etc/shadow` (root-only) holds hashes.
- `unshadow` merges the two for JtR/hashcat.
- Hash format: `$<id>$<salt>$<hash>` — the `<id>` tells you the algorithm.
- Modern Debian uses `yescrypt` (`$y$`) by default; older systems use `sha512crypt` (`$6$`).
- `john --single` is *uniquely* suited to Linux shadow hashes because GECOS gives candidate words.

## `/etc/passwd` Layout
Format: 7 colon-separated fields per user.
```
htb-student:x:1000:1000:,,,:/home/htb-student:/bin/bash
```
| # | Field | Example |
|---|-------|---------|
| 1 | Username | `htb-student` |
| 2 | Password placeholder | `x` (means hash is in `/etc/shadow`) |
| 3 | UID | `1000` |
| 4 | GID | `1000` |
| 5 | GECOS (real name, room, phone) | `,,,` |
| 6 | Home directory | `/home/htb-student` |
| 7 | Login shell | `/bin/bash` |

### Misconfiguration: writable `/etc/passwd`
If `/etc/passwd` is writable by an unprivileged user, you can blank the root password field:
```
root::0:0:root:/root:/bin/bash
```
Then `su` walks straight in without a prompt. Rare but devastating.

## `/etc/shadow` Layout
9 colon-separated fields. Hash field uses `$id$salt$hash`.
```
htb-student:$y$j9T$3QSBB...:18955:0:99999:7:::
```
| # | Field |
|---|-------|
| 1 | Username |
| 2 | Password hash (`$id$salt$hash`) |
| 3 | Days since 1970 of last password change |
| 4 | Minimum age |
| 5 | Maximum age |
| 6 | Warn period |
| 7 | Inactivity period |
| 8 | Expiration date |
| 9 | Reserved |

### Hash Algorithm IDs

| ID | Algorithm |
|----|-----------|
| `$1$` | MD5 |
| `$2a$` / `$2b$` / `$2y$` | bcrypt |
| `$5$` | SHA-256 crypt |
| `$6$` | SHA-512 crypt |
| `$sha1$` | SHA1 crypt |
| `$y$` / `$gy$` | yescrypt (Debian 11+ default) |
| `$7$` | scrypt |

Special non-hash values in field 2:
- `*` or `!` — user can't log in via password (key auth or locked account still possible)
- *empty* — passwordless login (rare; usually misconfiguration)

## `/etc/security/opasswd` — Old Password History
PAM's password history file (used by `pam_unix`'s `remember=N` option):
```bash
sudo cat /etc/security/opasswd
```
```
cry0l1t3:1000:2:$1$HjFAfYTG$qNDkF...,$1$kcUjWZJX$E9uMSm...
```
Each entry is a previously-used password hash. Useful for password-pattern analysis (users often increment one digit when rotating).

## Cracking Workflow

### Unshadow
```bash
sudo cp /etc/passwd /tmp/passwd.bak
sudo cp /etc/shadow /tmp/shadow.bak
unshadow /tmp/passwd.bak /tmp/shadow.bak > /tmp/unshadowed.hashes
```

### With Hashcat (mode 1800 = sha512crypt; auto for others)
```bash
hashcat -m 1800 -a 0 /tmp/unshadowed.hashes /usr/share/wordlists/rockyou.txt
```

### With JtR
```bash
john /tmp/unshadowed.hashes
john /tmp/unshadowed.hashes --show
```
For yescrypt (`$y$`), JtR-jumbo auto-detects. Hashcat mode for yescrypt: `29800`.

### Single-crack mode
Especially powerful here because the GECOS field gives candidate words tailored to each user:
```bash
john --single /tmp/unshadowed.hashes
```

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — Use single-crack to find Martin's password | **(hidden — see HTB walkthrough)** | Extract `martin:` line into a hash file, `john --single hash.txt` |
| Q2 — Use a wordlist attack to find Sarah's password | **(hidden)** | Extract `sarah:` line, `hashcat -m 1800 hash rockyou.txt` |

## Key Takeaways
- `/etc/shadow` is the prize — readable only by root. Often grabbed via LFI, NFS misconfig, or backup exposure.
- `$y$` (yescrypt) is the new default on modern Debian/Ubuntu. Slower than `$6$` to crack.
- `unshadow` produces input suitable for both JtR and Hashcat — same file works for either.
- Single-crack mode shines on Linux because the GECOS field gives username-based candidates *per user*.

## Gotchas
- `$2a$` is bcrypt — *very* slow on hashcat. Don't waste cycles on it without a small targeted wordlist.
- `*` and `!` in the password field do NOT mean "no password". They mean "no password-based login allowed" — keys may still work.
- Older systems used DES crypt (no `$` prefix, 13 chars). Max password length 8 — trivially crackable.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[15-credential-hunting-in-windows]] | [[17-credential-hunting-in-linux]] →
<!-- AUTO-LINKS-END -->
