# LAB — Sub-domain Fuzzing

## ID
7

## Module
Ffuf

## Kind
lab

## Title
Section 7 — Sub-domain Fuzzing

## Description
Use ffuf with the FUZZ keyword in the hostname slot to discover **public** sub-domains via DNS resolution.

## Tags
ffuf, sub-domain, dns, lab, public-dns

## Commands
- `ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u https://FUZZ.inlanefreight.com/`
- `ffuf -s -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u 'http://FUZZ.inlanefreight.com/'`

## What This Section Covers
Sub-domain fuzzing only finds sub-domains that **have a public DNS record**. The lookup goes through your resolver — if the resolver doesn't know about it, ffuf can't reach it. For lab targets without public DNS, this method silently returns nothing useful (errors only).

## Methodology
1. Bind FUZZ to a sub-domain wordlist:
   `-w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ`
2. Place FUZZ in the hostname slot of the URL: `-u https://FUZZ.example.com/`
3. Run ffuf. Hits = sub-domains that resolve and return matching status codes.
4. For larger coverage, swap to `subdomains-top1million-110000.txt`.

## When sub-domain fuzzing fails
- Target has **no public DNS records** for sub-domains (typical of HTB lab boxes).
- All requests error out (DNS resolution failure) — output looks like `Errors: 4997`.
- Solution: switch to **vhost fuzzing** (next section) — it bypasses DNS by setting the `Host:` header directly against a known IP.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Customer sub-domain on `inlanefreight.com` | **customer.inlanefreight.com** | `ffuf -s -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u 'http://FUZZ.inlanefreight.com/'` returned `customer` among other public subdomains (`www`, `blog`, `support`, `ns3`, `my`) |

## Key Takeaways
- Sub-domain fuzzing is a **public DNS** technique. It only sees what the public internet has registered.
- A wall of `Errors:` and `0 hits` against a lab domain is the signal to switch to vhost fuzzing.
- Treat sub-domain hits like new targets — each one may have its own page tree, vhost stack, and parameters.

## Gotchas
- HTTPS targets sometimes return TLS errors for non-existent sub-domains. ffuf's default error counter goes up; that's not a hit.
- Some CDNs (Cloudflare, etc.) return identical responses for non-existent sub-domains — filter by response size, not just status.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ffuf]]
← [[06-dns-records]] | [[08-vhost-fuzzing]] →
<!-- AUTO-LINKS-END -->
