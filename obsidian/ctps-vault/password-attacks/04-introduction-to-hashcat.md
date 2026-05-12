# NOTE — Introduction to Hashcat

## ID
505

## Module
Password Attacks

## Kind
notes

## Title
Section 4 — Introduction to Hashcat

## Description
GPU-accelerated cracking with hashcat: attack modes (dictionary, mask, rules-based), hash mode IDs, the built-in mask charsets, and the `best64.rule` ruleset.

## Tags
hashcat, gpu, dictionary-attack, mask-attack, rules, best64, hashid

## Commands
- `hashcat -a 0 -m <mode> <hash> <wordlist>`
- `hashcat -a 0 -m 0 hash.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule`
- `hashcat -a 3 -m 0 <hash> '?u?l?l?l?l?d?s'`
- `hashcat -m <mode> <hash> --show`
- `hashcat --help | grep -i <hash_name>`
- `hashid -m <hash>`

## Concept Overview
Hashcat is the de-facto password cracker on systems with a GPU. Supports hundreds of hash types via numeric IDs (`-m`) and multiple attack modes (`-a`). General syntax:
```
hashcat -a <attack_mode> -m <hash_type> <hash_or_file> [wordlist|mask|rules]
```

## Attack Modes (`-a`)

| Mode | Name | Description |
|------|------|-------------|
| `-a 0` | **Dictionary** | Tries each line of a wordlist (optionally with `-r <rule>`) |
| `-a 1` | **Combinator** | Concatenates entries from two wordlists |
| `-a 3` | **Mask** | Brute-force using a defined character pattern |
| `-a 6` | **Hybrid (wordlist + mask)** | e.g. `password` → `password!`, `password!!` |
| `-a 7` | **Hybrid (mask + wordlist)** | e.g. `password` → `1password`, `12password` |
| `-a 9` | **Association** | Associates wordlist entries with corresponding hashes |

## Common Hash Modes (`-m`)

| Mode | Hash Type |
|------|-----------|
| `0` | MD5 |
| `100` | SHA1 |
| `1400` | SHA-256 |
| `1700` | SHA-512 |
| `1000` | NTLM |
| `3000` | LM |
| `500` | MD5 Crypt / FreeBSD MD5 / Cisco-IOS |
| `1800` | sha512crypt (Linux `$6$`) |
| `5200` | Password Safe v3 |
| `5500` | NetNTLMv1 |
| `5600` | NetNTLMv2 |
| `13100` | Kerberos 5 TGS-REP (Kerberoasting) |
| `18200` | Kerberos 5 AS-REP (AS-REP roasting) |
| `2100` | DCC2 (Domain Cached Credentials v2) |
| `22100` | BitLocker |

Use `hashcat --help | grep -i <type>` or [hashcat example hashes](https://hashcat.net/wiki/doku.php?id=example_hashes) when unsure.

## Mask Attack Charsets (`-a 3`)

| Symbol | Charset |
|--------|---------|
| `?l` | a–z |
| `?u` | A–Z |
| `?d` | 0–9 |
| `?h` | 0–9 a–f (lowercase hex) |
| `?H` | 0–9 A–F (uppercase hex) |
| `?s` | special chars (`!"#$%&'()*+,...`) |
| `?a` | `?l?u?d?s` (all printable ASCII) |
| `?b` | 0x00–0xff (full byte range) |

Custom charsets via `-1 <chars>` … `-4 <chars>`, then reference with `?1`–`?4`.

Example — uppercase + 4 lowercase + digit + symbol (8 chars):
```bash
hashcat -a 3 -m 0 1e293d6912d074c0fd15844d803400dd '?u?l?l?l?l?d?s'
```

## Rules — `best64.rule`
Built-in ruleset that applies ~64 common mutations (append digits, leet-speak, capitalize). High-yield with low cost:
```bash
hashcat -a 0 -m 0 hash.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```
Other rules on Pwnbox: `dive.rule`, `T0XlC.rule`, `unix-ninja-leetspeak.rule`, `rockyou-30000.rule`.

## Showing Already-Cracked Hashes
Hashcat saves cracked results to `~/.hashcat/hashcat.potfile`:
```bash
hashcat -m 0 <hash> --show
```

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — Dictionary crack `e3e3ec5831ad5e7288241960e5d4fdb8` | **(hidden)** | `hashcat -a 0 -m 0 e3e3ec5831ad5e7288241960e5d4fdb8 rockyou.txt` |
| Q2 — Dictionary + rules crack `1b0556a75770563578569ae21392630c` | **(hidden)** | `hashcat -a 0 -m 0 ... rockyou.txt -r best64.rule` |
| Q3 — Mask crack `1e293d6912d074c0fd15844d803400dd` | **(hidden)** | `hashcat -a 3 -m 0 ... '?u?l?l?l?l?d?s'` |

## Key Takeaways
- `-m 1000` = NTLM, the format used by Windows for local + domain accounts. Memorize it.
- Mask attacks are *much* faster than naïve brute-force when you know the password structure (length, char categories).
- Hashcat will hit "wordlist too small" warnings on tiny wordlists — combine with rules to amplify.
- Always start with dictionary → dictionary+rules → mask → exhaustive incremental.

## Gotchas
- On Pwnbox/VMs without GPU passthrough, hashcat falls back to CPU — much slower than native GPU.
- Mode IDs change occasionally between major versions. Check `hashcat --help` if the mode you remember stops working.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[03-introduction-to-john-the-ripper]] | [[05-writing-custom-wordlists-and-rules]] →
<!-- AUTO-LINKS-END -->
