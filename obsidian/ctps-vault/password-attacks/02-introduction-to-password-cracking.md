# NOTE — Introduction to Password Cracking

## ID
503

## Module
Password Attacks

## Kind
theory

## Title
Section 2 — Introduction to Password Cracking

## Description
Hashing fundamentals (MD5/SHA-256), salting to defeat rainbow tables, and the three core cracking techniques: rainbow tables, brute-force, and dictionary attacks.

## Tags
password-cracking, hashing, salt, rainbow-tables, brute-force, dictionary-attack, rockyou, theory

## Commands
- `echo -n Soccer06! | md5sum`
- `echo -n Soccer06! | sha256sum`
- `echo -n Th1sIsTh3S@lt_Soccer06! | md5sum`
- `head --lines=20 /usr/share/wordlists/rockyou.txt`

## TL;DR — What's Important
- Hashes are one-way functions; "cracking" = guess-and-hash, not reverse.
- Salt = random bytes prepended/appended before hashing → kills generic rainbow tables.
- A 1-byte salt multiplies rainbow-table size by 256.
- Dictionary attacks beat brute-force in real-world pentests; brute-force is for short passwords only.

## Concept Overview
Passwords are stored as hashes so a DB leak doesn't immediately expose plaintext. Common algorithms: MD5, SHA-256, SHA-512, NTLM, bcrypt. Cracking = generating candidate strings, hashing them, and comparing to the target hash.

## Cracking Techniques

| Technique | How it Works | When to Use |
|-----------|--------------|-------------|
| **Rainbow tables** | Pre-computed hash→password lookup tables | Fast on unsalted hashes (LM, raw MD5/SHA1) |
| **Brute-force** | Try every char combo — 100% effective but slow | Short passwords (<9 chars), simple charsets |
| **Dictionary attack** | Test passwords from a wordlist (rockyou, SecLists) | Default first choice — fast, hits ~80% of real-world passes |

## Speed Reference
- MD5 on a typical laptop: ~5M guesses/sec
- DCC2 hash on same hardware: ~10K guesses/sec
- Speed depends *heavily* on algorithm + hardware (GPU > CPU)

## Salting
```
hash(password)             ← rainbow table can hit
hash(salt || password)     ← rainbow table useless unless rebuilt per salt
```
Salt is **not secret** — it's stored with the hash so the auth system can recompute. Its only job is breaking pre-computation attacks.

## Key Takeaways
- A "fast hash" (MD5, SHA1) being cracked quickly is a *design* problem — modern algorithms (bcrypt, argon2) intentionally slow down hashing to penalize attackers.
- rockyou.txt = 14M real leaked passwords from 2009. Still effective today because humans reuse patterns.
- Unsalted hashes + weak algorithm = `crackstation.net` may already have it.

## Gotchas
- "Cracking 100% of hashes" is impossible against strong passwords + slow hashes within typical engagement timelines.
- Brute-force is *not* the default approach — it's the last resort.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[00-overview]] | [[03-introduction-to-john-the-ripper]] →
<!-- AUTO-LINKS-END -->
