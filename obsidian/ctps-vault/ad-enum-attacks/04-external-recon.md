# THEORY — External Recon and Enumeration Principles

## ID
703

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 4 — External Recon and Enumeration Principles

## Description
Learn how to perform passive external reconnaissance before an Active Directory penetration test: finding IP space, DNS records, usernames, leaked credentials, and other publicly accessible data that can inform internal testing.

## Tags
external-recon, osint, enumeration, theory, dns

## TL;DR — What's Important
- **Start passive, stay wide** – gather everything from public sources before any active scanning. This validates scope and finds low‑hanging fruit.
- **DNS records often leak internal info** – TXT records, SPF, DMARC, and misconfigured zone transfers can reveal subdomains and internal IPs.
- **Email format = username format** – contact pages, LinkedIn, and news articles give you naming conventions (e.g., `first.last`, `flast`). Use these to build spraying lists.
- **Breach data is gold** – platforms like Dehashed let you check if the target’s emails/passwords have been leaked. Try those creds against VPN, OWA, or RDP.
- **Job postings reveal tech stacks** – companies list required skills (e.g., “experience with AWS, Azure AD, CrowdStrike”). That’s your attack surface map.

## Concept Overview
External enumeration (also called OSINT – open source intelligence) is the phase before any active testing. You collect information about the target from publicly available sources: DNS, WHOIS, ASN registries, search engines, social media, code repositories, and breach databases. The goal is to map the target’s external footprint, discover username schemas, identify exposed services, and find leaked credentials – all without sending a single packet to the target’s infrastructure. This passive approach is legal, stealthy, and often yields the first foothold (e.g., a password from a breach that works on the VPN). This section covers the methodology, tools, and data points you need to collect before moving to active enumeration.

## Key Concepts

### What to Collect

| Data Point | Why It Matters |
|------------|----------------|
| IP space (ASN, netblocks) | Defines external attack surface |
| DNS records (A, MX, NS, TXT, SPF, DMARC) | Subdomains, mail servers, internal IPs, TXT flags |
| Email / username format | Build username lists for spraying |
| Job postings | Tech stack, software versions, cloud providers |
| Public documents (PDF, DOCX, PPT) | Embedded metadata, intranet links, internal hostnames |
| Leaked credentials (breaches) | Valid passwords to try against external auth portals |
| GitHub / cloud storage | Keys, secrets, config files |

### Key Tools & Resources

| Category | Tools / Resources |
|----------|-------------------|
| ASN / IP | IANA, ARIN, RIPE, BGP Toolkit (`bgp.he.net`) |
| DNS | `dig`, `nslookup`, `host`, ViewDNS.info, PTRArchive |
| Search engines | Google Dorks, Bing, Baidu |
| Social media | LinkedIn, Twitter, Facebook, news articles |
| Username harvesting | `linkedin2username` |
| Breach data | HaveIBeenPwned, Dehashed (paid) |
| Code / cloud | Trufflehog, Greyhat Warfare |

### DNS Enumeration Commands

```bash
# Query all record types
dig any inlanefreight.com
dig txt inlanefreight.com
dig mx inlanefreight.com
dig ns inlanefreight.com
dig a inlanefreight.com

# Alternatives (if dig not available)
nslookup -type=txt inlanefreight.com
nslookup -type=any inlanefreight.com
host -t txt inlanefreight.com

# Validate nameservers
nslookup ns1.inlanefreight.com
nslookup ns2.inlanefreight.com

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[03-scenario]] | [[05-initial-domain-enum]] →
<!-- AUTO-LINKS-END -->
