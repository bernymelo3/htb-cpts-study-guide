# NOTE — Subdomain Bruteforcing

## ID
206

## Module
Information Gathering – Web Edition

## Kind
lab

## Title
Section 7 — Subdomain Bruteforcing

## Description
Active subdomain discovery via wordlist brute-force using `dnsenum` (with optional Google scraping, recursion, and zone-transfer attempts). Demonstrates the SecLists top-20000 wordlist against `inlanefreight.com`.

## Tags
subdomain, brute-force, dnsenum, fierce, dnsrecon, amass, assetfinder, puredns, seclists

## Commands
- `dnsenum --enum inlanefreight.com -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -r`
- `dnsenum --enum inlanefreight.com -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt`

## What This Section Covers
The four-step brute-force workflow: choose wordlist → iterate → DNS-query each candidate → validate. Plus a tour of the major subdomain brute-force tools and a hands-on `dnsenum` demo against the lab target.

## Methodology
1. **Pick a wordlist** — general (SecLists top-1M), targeted (industry/tech-specific), or custom (built from OSINT).
2. **Iterate**: tool appends each word to the base domain → `dev.example.com`, `staging.example.com`, ...
3. **DNS-query** each candidate (A or AAAA).
4. **Validate**: resolved → add to live list. Optional: check it loads in a browser (kills wildcards/parking).

## Tool Comparison
| Tool | Highlight |
|---|---|
| `dnsenum` | Perl, multi-purpose: brute force + zone transfer + Google scraping + reverse lookups |
| `fierce` | Wildcard detection, recursive, beginner-friendly |
| `dnsrecon` | Multiple techniques in one + customisable output formats |
| `amass` | OWASP project — heavy passive sources, integrates with everything |
| `assetfinder` | Lightweight, fast — single binary |
| `puredns` | High-performance brute-forcer with custom resolver lists, wildcard filtering |

## DNSEnum — what it does
1. **DNS record enumeration** (A, AAAA, NS, MX, TXT)
2. **Zone transfer attempt** against discovered NS
3. **Subdomain brute force** with `-f <wordlist>`
4. **Google scraping** for additional subdomains
5. **Reverse lookups** on discovered IPs
6. **WHOIS** lookup for ownership info

## Demo Command Breakdown
```bash
dnsenum --enum inlanefreight.com \
  -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  -r
```
| Flag | Meaning |
|---|---|
| `--enum` | Shortcut bundle (threads, brute, etc.) |
| `-f <path>` | Wordlist for brute force |
| `-r` | Recurse — brute-force subdomains *of* discovered subdomains |

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Find the missing subdomain of `inlanefreight.com` (known: www, ns1, ns2, ns3, blog, support, customer) | **{see lab — answer redacted in walkthrough}** | `dnsenum --enum inlanefreight.com -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt` — examine output for the unfamiliar entry |

## Key Takeaways
- Wordlist quality > tool choice. SecLists `subdomains-top1million-20000.txt` is the default starting point.
- `-r` recursive can multiply runtime hugely — enable only when the surface looks deep (e.g. `dev.example.com` exists, brute-force its subs too).
- Brute-force is loud at the DNS layer — your queries land in the target's authoritative NS logs.

## Gotchas
- Wildcard DNS (`*.example.com → 1.2.3.4`) makes everything appear to resolve. Always validate with HTTP fetch or check the IP against the wildcard's IP.
- `dnsenum`'s Google-scraping module gets rate-limited fast — add a sleep or skip it (`--noreverse` etc.).
- Make sure the SecLists path matches your install (`/usr/share/seclists` on Pwnbox/Kali; may differ on other distros).

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← [[06-subdomains]] | [[08-dns-zone-transfers]] →
<!-- AUTO-LINKS-END -->
