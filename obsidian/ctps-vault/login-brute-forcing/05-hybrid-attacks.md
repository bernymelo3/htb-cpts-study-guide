# NOTE — Hybrid Attacks

## ID
512

## Module
Password Attacks

## Kind
notes

## Title
Section 5 — Hybrid Attacks (Wordlist Filtering & Credential Stuffing)

## Description
Learn how hybrid attacks combine dictionary and brute‑force techniques, then practice filtering a password wordlist against a real password policy using `grep` and regex.

## Tags
hybrid-attacks, wordlist-filtering, grep, regex, credential-stuffing

## Commands
- `wget https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/darkweb2017_top-10000.txt`
- `grep -E '^.{8,}$' darkweb2017_top-10000.txt > darkweb2017-minlength.txt`
- `grep -E '[A-Z]' darkweb2017-minlength.txt > darkweb2017-uppercase.txt`
- `grep -E '[a-z]' darkweb2017-uppercase.txt > darkweb2017-lowercase.txt`
- `grep -E '[0-9]' darkweb2017-lowercase.txt > darkweb2017-number.txt`
- `wc -l darkweb2017-number.txt`

## What This Section Covers
Hybrid attacks exploit predictable password modifications (e.g., `Summer2023` → `Summer2024!`). This section explains how attackers combine dictionary and brute‑force techniques. It then walks through filtering a real wordlist (`darkweb2017_top-10000.txt`) against a password policy (min length 8, uppercase, lowercase, digit) using `grep` with regular expressions. Finally, it introduces credential stuffing as a related attack that leverages password reuse across services.

## Methodology
1. **Download a wordlist** — use `wget` to fetch `darkweb2017_top-10000.txt` from SecLists.
2. **Filter for minimum length (8+ characters)** — `grep -E '^.{8,}$'` keeps passwords with at least 8 characters.
3. **Filter for at least one uppercase letter** — `grep -E '[A-Z]'`.
4. **Filter for at least one lowercase letter** — `grep -E '[a-z]'`.
5. **Filter for at least one digit** — `grep -E '[0-9]'`.
6. **Count the results** — `wc -l` shows how many passwords survive all filters (typically ~89 out of 10,000).

## Multi-step Workflow
```bash
# Download the wordlist
wget https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/darkweb2017_top-10000.txt

# Filter by minimum length (8 characters)
grep -E '^.{8,}$' darkweb2017_top-10000.txt > darkweb2017-minlength.txt

# Filter by at least one uppercase letter
grep -E '[A-Z]' darkweb2017-minlength.txt > darkweb2017-uppercase.txt

# Filter by at least one lowercase letter
grep -E '[a-z]' darkweb2017-uppercase.txt > darkweb2017-lowercase.txt

# Filter by at least one digit
grep -E '[0-9]' darkweb2017-lowercase.txt > darkweb2017-number.txt

# See how many passwords made it through
wc -l darkweb2017-number.txt

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|login-brute-forcing]]  
← [[04-dictionary-attacks]] | [[06-hydra]] →
<!-- AUTO-LINKS-END -->
