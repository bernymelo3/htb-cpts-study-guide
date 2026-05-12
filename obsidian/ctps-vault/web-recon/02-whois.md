# NOTE — WHOIS

## ID
201

## Module
Information Gathering – Web Edition

## Kind
theory

## Title
Section 2 — WHOIS

## Description
WHOIS is a public lookup protocol for domain registration metadata — registrar, registrant, dates, name servers, abuse contact. It's free pre-engagement intel that hints at infrastructure and personnel.

## Tags
whois, passive-recon, registrar, registrant, name-server, iana

## TL;DR — What's Important
- WHOIS = public phonebook for the internet (domains, IP blocks, ASNs).
- Each record contains: domain, registrar, registrant/admin/tech contact, creation/expiry dates, name servers.
- For recon: identifies key personnel, hints at hosting infra, history reveals ownership changes.
- Many records are now privacy-shielded — but expiry dates, registrar, and name servers always leak.

## Concept Overview
The WHOIS protocol queries databases held by registries and registrars. Originating with Elizabeth Feinler at SRI's NIC (1970s ARPANET), it's still the canonical way to look up who registered a domain and where its DNS authority lives.

## What's in a WHOIS Record
| Field | Meaning |
|---|---|
| Domain Name | The registered domain |
| Registrar | Company where the domain was registered (GoDaddy, Namecheap, RegistrarSafe, Amazon...) |
| Registrar IANA ID | Numeric ID identifying the registrar — useful for fingerprinting |
| Registrant Contact | Who owns the domain (often privacy-protected) |
| Admin Contact | Manages the domain |
| Tech Contact | Handles technical issues |
| Creation / Expiry Date | Lifecycle of the registration |
| Name Servers | NS records — hint at hosting provider |
| Abuse Contact | Email/phone to report abuse |

## Why It Matters in Recon
1. **Identifying key personnel** — names, emails, phone numbers usable for OSINT/phishing pretexts.
2. **Discovering network infrastructure** — name servers + IPs hint at hosting/DNS provider.
3. **Historical analysis** — tools like WhoisFreaks track changes over time (ownership transfers, NS changes).
4. **Trust scoring** — recently-registered domain + privacy-shielded registrant + bulletproof host = phishing red flag.

## Example — Truncated WHOIS for `inlanefreight.com`
```
Domain Name: inlanefreight.com
Registry Domain ID: 2420436757_DOMAIN_COM-VRSN
Registrar WHOIS Server: whois.registrar.amazon
Registrar URL: https://registrar.amazon.com
Updated Date: 2023-07-03T01:11:15Z
Creation Date: 2019-08-05T22:43:09Z
```

## Key Takeaways
- WHOIS is purely passive — the target never sees your query.
- Privacy services hide registrant data, but `Updated`, `Created`, `Expiry`, registrar, and name servers always remain.
- Combine WHOIS with DNS + crt.sh — together they paint the full external picture.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← [[01-introduction]] | [[03-utilizing-whois]] →
<!-- AUTO-LINKS-END -->
