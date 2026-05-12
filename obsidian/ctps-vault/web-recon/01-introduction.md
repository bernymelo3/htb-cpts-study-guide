# NOTE — Introduction

## ID
200

## Module
Information Gathering – Web Edition

## Kind
theory

## Title
Section 1 — Introduction

## Description
Frames web reconnaissance as the first paid hour of every engagement: identify assets, discover hidden information, map the attack surface, and gather intelligence — split into active vs. passive techniques.

## Tags
web-recon, methodology, active-recon, passive-recon, osint, attack-surface

## TL;DR — What's Important
- **Goals of web recon:** identify assets, discover hidden info, analyse attack surface, gather intel.
- **Active = direct interaction** with the target (port scan, banner grab, vuln scan, crawl) — louder, more complete, more likely to trigger IDS/WAF.
- **Passive = no packets to target** (search engines, WHOIS, DNS, CT logs, social media, GitHub) — stealthy but less complete.
- Real engagements blend both: passive first to scope cheaply, active second to fill gaps once scope is confirmed.

## Concept Overview
Web reconnaissance is the systematic collection of information about a target website or application. It maps to the "Information Gathering" phase of a pentest and feeds every subsequent phase (vuln assessment, exploitation, post-ex). Defenders use the same techniques to find what attackers will see.

## Active Reconnaissance — Common Techniques
| Technique | What it does | Example tools | Detection risk |
|---|---|---|---|
| Port scanning | Identify open ports/services | Nmap, Masscan | High |
| Vulnerability scanning | Probe for known CVEs/misconfigs | Nessus, OpenVAS, Nikto | High |
| Network mapping | Map topology + hops | Traceroute, Nmap | Medium-High |
| Banner grabbing | Pull service banners | Netcat, curl | Low |
| OS fingerprinting | Identify host OS | Nmap `-O`, Xprobe2 | Low |
| Service enumeration | Identify service version | Nmap `-sV` | Low |
| Web spidering | Crawl the site | Burp Spider, ZAP, Scrapy | Low-Medium |

## Passive Reconnaissance — Common Techniques
| Technique | What it does | Example tools | Detection risk |
|---|---|---|---|
| Search engine queries | Pull indexed data | Google, Bing, Shodan, DuckDuckGo | Very low |
| WHOIS lookups | Domain registration data | `whois`, online lookups | Very low |
| DNS analysis | Subdomains, mail, NS, TXT | `dig`, `nslookup`, `dnsenum` | Very low |
| Web archive analysis | Historical snapshots | Wayback Machine | Very low |
| Social media analysis | People + roles | LinkedIn, Twitter, OSINT tools | Very low |
| Code repos | Leaked secrets/source | GitHub, GitLab | Very low |

## Key Takeaways
- Treat passive recon as "free intel" — do it first, it costs you nothing in noise.
- Active recon is where you tip your hand. Save it for after you've confirmed scope.
- The split is a continuum, not a binary — banner grabbing is "active" but barely visible; CT log scraping is "passive" but explicit.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← *(start of module)* | [[02-whois]] →
<!-- AUTO-LINKS-END -->
