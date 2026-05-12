# NOTE — Subdomains

## ID
205

## Module
Information Gathering – Web Edition

## Kind
theory

## Title
Section 6 — Subdomains

## Description
Why subdomains matter (dev/staging envs, hidden admin panels, legacy apps, accidental data exposure) and the two enumeration approaches: active (zone transfer, brute force) vs passive (CT logs, search engines, DNS aggregators).

## Tags
subdomains, theory, enumeration, active-recon, passive-recon, ct-logs, dev-staging

## TL;DR — What's Important
- Subdomains often expose dev/staging envs, admin panels, legacy apps, leaked docs.
- DNS-wise: subdomains are A/AAAA records (sometimes CNAMEs). Discovery = enumerate those records.
- **Active enumeration** = brute force or zone transfer — direct DNS queries, more complete, more detectable.
- **Passive enumeration** = CT logs, search engines, OSINT aggregators — stealthy, less complete.
- Real engagements combine both.

## Concept Overview
A subdomain is anything to the left of the main domain (`blog.example.com`, `dev.example.com`). They're often created for organisational/operational reasons but become the soft underbelly of an external attack surface — security on `dev.` is almost never as tight as on `www.`.

## Why Subdomains Matter
| Subdomain type | What it might leak |
|---|---|
| Dev / staging | Unpatched versions, debug endpoints, default creds |
| Admin / management | `admin.`, `manage.`, `panel.` — login pages not meant to be public |
| Legacy apps | Old PHP versions, EOL software, known CVEs |
| Sensitive | Internal docs, configs, file shares accidentally exposed |

## Two Enumeration Approaches
### 1. Active — direct DNS queries
- **Zone transfer** ([[08-dns-zone-transfers]]): if a misconfigured NS allows AXFR, you get *all* records in one query.
- **Brute force** ([[07-subdomain-bruteforcing]]): test a wordlist of guesses against the zone (`dnsenum`, `ffuf`, `gobuster`, `amass`).

### 2. Passive — third-party data
- **CT logs** ([[10-certificate-transparency-logs]]): every cert issued for the domain is publicly logged with its SAN list.
- **Search engines**: `site:*.example.com` filters indexed pages.
- **OSINT aggregators**: SecurityTrails, VirusTotal, Shodan, AnubisDB.

## Active vs Passive Trade-off
| | Active | Passive |
|---|---|---|
| Detectability | DNS server may log queries | Target sees nothing |
| Completeness | Limited by wordlist quality | Limited to what's been logged/indexed |
| Speed | Slower (DNS rate limits) | Instant (already aggregated) |
| Coverage | Live records only | Includes expired/dead subdomains |

## Key Takeaways
- Always run passive first — it's free and the target doesn't see it.
- Then brute-force with a quality wordlist (SecLists `subdomains-top1million-*`).
- Cross-check the union — passive will find expired ones, active will find live ones the world hasn't seen.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← [[05-digging-dns]] | [[07-subdomain-bruteforcing]] →
<!-- AUTO-LINKS-END -->
