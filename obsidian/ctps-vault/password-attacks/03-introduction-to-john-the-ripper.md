# NOTE — Introduction to John The Ripper

## ID
504

## Module
Password Attacks

## Kind
notes

## Title
Section 3 — Introduction to John The Ripper

## Description
JtR cracking modes (single, wordlist, incremental), hash identification with hashID, and the `2john` helper family for converting files (SSH, PDF, ZIP, etc.) to crackable hashes.

## Tags
john-the-ripper, jtr, single-crack, wordlist-mode, incremental, hashid, 2john, ssh2john

## Commands
- `john --single <hash_file>`
- `john --wordlist=<wordlist> <hash_file>`
- `john --wordlist=<wordlist> --rules <hash_file>`
- `john --incremental <hash_file>`
- `john --format=<format> --wordlist=<wordlist> <hash_file>`
- `john <hash_file> --show`
- `hashid -j <hash>`
- `ssh2john id_rsa > ssh.hash`

## Concept Overview
John the Ripper (JtR / john) is an open-source offline password cracker. Use the **jumbo** variant (`john-jumbo`) — it supports far more hash formats and rules than vanilla. JtR auto-detects most hash formats but `--format=<x>` forces it when ambiguous.

## Cracking Modes

| Mode | Flag | What It Does |
|------|------|--------------|
| **Single crack** | `--single` | Mutates the username + GECOS (real name, home dir) — best for `/etc/shadow` style hashes where the user info is in the same file |
| **Wordlist** | `--wordlist=<file>` | Tries each line of a wordlist; pair with `--rules` for mutations |
| **Incremental** | `--incremental` | Brute-force using Markov chains (statistical model) — exhaustive but slow |

### Single-Crack Example
Given an `/etc/passwd`+`/etc/shadow` line for user `r0lf` with GECOS `Rolf Sebastian`:
```bash
john --single hash.txt
```
JtR generates candidates like `r0lf`, `Rolf`, `Sebastian1`, `naitsabeSr`, etc.

### Wordlist + Rules
```bash
john --wordlist=rockyou.txt --rules ssh.hash
```
Rules apply transformations (append digits, leet-speak, capitalize) on the fly.

## Hash Identification

`hashID` suggests possible formats + the JtR format string:
```bash
hashid -j 193069ceb0461e1d40d216e32c79c704
```
Output includes `[JtR Format: ripemd-128]`, `[JtR Format: nt]`, etc. Multiple matches are common — context (where the hash came from) is often what decides.

### Common `--format` Values

| Hash | JtR Format |
|------|------------|
| Raw MD5 | `raw-md5` |
| Raw SHA-1 | `raw-sha1` |
| Raw SHA-256 | `raw-sha256` |
| NTLM | `nt` |
| LM (legacy Windows) | `lm` |
| Linux SHA-512 crypt | (auto-detected from `$6$`) |
| RIPEMD-128 | `ripemd-128` |
| Domain Cached Credentials v2 | `mscash2` |
| Kerberos 5 | `krb5` |
| ZIP archive | `zip` |
| SSH private key | `ssh` |

## The `2john` Helper Family
Convert protected files into JtR-compatible hashes:

| Tool | Target File |
|------|-------------|
| `ssh2john` | SSH private keys (encrypted) |
| `zip2john` | ZIP archives |
| `pdf2john` | PDFs |
| `office2john` | MS Office docs (.docx, .xlsx) |
| `keepass2john` | KeePass .kdbx databases |
| `rar2john` | RAR archives |
| `bitlocker2john` | BitLocker volumes |
| `7z2john.pl` | 7-Zip archives |

Pattern:
```bash
<tool> <protected_file> > file.hash
john --wordlist=rockyou.txt file.hash
john file.hash --show
```

Find all on Pwnbox:
```bash
locate *2john*
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Use single-crack mode to crack r0lf's password | **(hidden — see HTB walkthrough)** | `john --single hash.txt` where hash.txt has the full passwd-format line |
| Q2 — Use wordlist mode with rockyou.txt to crack the RIPEMD-128 password | **(hidden)** | `john --format=ripemd-128 --wordlist=rockyou.txt hash.txt` |

## Key Takeaways
- `--show` re-displays already-cracked hashes from `~/.john/john.pot` — no re-crack needed.
- Single-crack mode is criminally underused; if you have `passwd`+`shadow` together, run it first.
- `john-jumbo` ≠ vanilla john. Always check `john --list=formats | wc -l` to confirm you have jumbo.
- `unshadow passwd shadow > combined` merges the two files into JtR-friendly format.

## Gotchas
- JtR caches cracked passwords in `~/.john/john.pot`. Delete it if you want to re-crack from scratch.
- `--format` is case-sensitive. `nt` ≠ `NT` in some versions.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[02-introduction-to-password-cracking]] | [[04-introduction-to-hashcat]] →
<!-- AUTO-LINKS-END -->
