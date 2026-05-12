# NOTE — Writing Custom Wordlists and Rules

## ID
506

## Module
Password Attacks

## Kind
methodology

## Title
Section 5 — Writing Custom Wordlists and Rules

## Description
Build targeted wordlists from OSINT, mutate them with custom hashcat rules, and use CeWL to scrape a company website for likely password keywords.

## Tags
custom-wordlist, rules, osint, cewl, hashcat-rules, password-mutation

## Commands
- `hashcat --force password.list -r custom.rule --stdout | sort -u > mut_password.list`
- `cewl https://www.example.com -d 4 -m 6 --lowercase -w wordlist.txt`
- `hashcat -a 0 -m 0 <hash> mut_password.list`

## Concept Overview
Real users follow predictable patterns: `<Word><Year><Symbol>` (e.g. `Inlanefreight2024!`). When OSINT reveals personal context (kids, pets, hobbies, city, employer), a small handcrafted base wordlist + aggressive rules will outperform `rockyou.txt -r best64.rule` for that target.

## Common Password Patterns
| Style | Example |
|-------|---------|
| Capitalize first letter | `Password` |
| Append numbers | `Password123` |
| Append current year | `Password2024` |
| Append month | `Password08` |
| Append `!` | `Password2024!` |
| Leet-speak swaps | `P@ssw0rd2024!` |

## Custom Rule Functions

| Function | What it does |
|----------|--------------|
| `:` | No-op (output as-is) |
| `l` | Lowercase all |
| `u` | Uppercase all |
| `c` | Capitalize first, lowercase rest |
| `C` | Lowercase first, uppercase rest |
| `t` | Toggle case |
| `sXY` | Replace all `X` with `Y` |
| `$!` | Append `!` |
| `$1$9$9$8` | Append `1998` (each char prefixed with `$`) |
| `^X` | Prepend `X` |

## Workflow

1. **Build base wordlist from OSINT:**
```bash
cat << EOF > password.list
Mark
White
August
1998
Nexura
Sanfrancisco
Bella
Maria
Alex
Baseball
EOF
```

2. **Write the rule file:**
```bash
cat << EOF > custom.rule
c
C
t
\$!
\$1\$9\$9\$8
\$1\$9\$9\$8\$!
sa@
so0
ss\$
EOF
```

3. **Generate the mutated list:**
```bash
hashcat --force password.list -r custom.rule --stdout | sort -u > mut_password.list
```

4. **Crack with it:**
```bash
hashcat -a 0 -m 0 97268a8ae45ac7d15c3cea4ce6ea550b mut_password.list
```

## CeWL — Scrape a Site for Keywords
Pulls candidate words from a target's web pages — perfect for building company-aware lists.
```bash
cewl https://www.inlanefreight.com -d 4 -m 6 --lowercase -w inlane.wordlist
```
- `-d 4` — crawl depth
- `-m 6` — minimum word length
- `--lowercase` — store lowercase
- `-w` — output file

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — Crack Mark White's password `97268a8ae45ac7d15c3cea4ce6ea550b` | **(hidden — see HTB walkthrough)** | Build OSINT-based base list + custom rules, then `hashcat -a 0 -m 0 <hash> mut_password.list` |

## Key Takeaways
- Combine OSINT (LinkedIn, company website, pet names, kids' names, birth years) with predictable mutation rules — this beats generic wordlists for targeted attacks.
- `--stdout` makes hashcat *generate candidates* without cracking — pipe to `sort -u` to get a clean wordlist.
- Build base list small (10–20 personal terms) and let rules explode it.
- The dollar sign in `cat << EOF` rules must be escaped (`\$`) to survive shell expansion.

## Gotchas
- A 100-line base list × 1000-rule file = 100K candidates. Watch list size on big rule files like `dive.rule` (800K+ rules).
- Save your mutated list to disk once — re-running mutation is slower than re-reading.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[04-introduction-to-hashcat]] | [[06-cracking-protected-files]] →
<!-- AUTO-LINKS-END -->
