# NOTE — Search Engine Discovery (Google Dorking)

## ID
215

## Module
Information Gathering – Web Edition

## Kind
theory

## Title
Section 16 — Search Engine Discovery

## Description
Use search engines as passive recon tools — `site:`, `inurl:`, `filetype:`, `intitle:`, `intext:`, `cache:`, `link:`, plus boolean and quote operators. Includes a curated set of Google Dork patterns for common pentest targets (login pages, exposed files, config files, DB backups).

## Tags
search-engine, google-dork, osint, passive-recon, site-operator, filetype, ghdb

## TL;DR — What's Important
- Search-engine queries with operators = "OSINT for free."
- `site:` scopes a search to one domain. `filetype:` finds documents. `inurl:` finds path patterns. `intitle:` finds page titles.
- Combined with boolean operators (`AND`, `OR`, `NOT`, `-`) and quotes for exact phrases.
- Google Hacking Database (GHDB) collects pre-built dorks for common findings.

## Concept Overview
Search engines have already indexed the public internet. Pentesters use them as a free OSINT layer — the target sees nothing because the queries hit Google/Bing/DuckDuckGo, not the target. Operator-driven queries narrow billions of results down to exactly what you're hunting for.

## Why It Works
- **Open**: legal and ethical — you're searching public data.
- **Broad**: search engines index a huge slice of the public web.
- **Easy**: no special tooling, just a browser.
- **Stealthy**: zero packets to the target.

## Limitations
- Indexes are incomplete — `noindex` directives, paywalls, dynamic content, and authenticated pages may not be there.
- Stale data — cached pages can be old; current site state may differ.
- Some sensitive content was indexed once, then removed — check `cache:` for snapshots.

## Search Operators — Full Cheat-Sheet
| Operator | What it does | Example |
|---|---|---|
| `site:` | Scope to a domain | `site:example.com` |
| `inurl:` | Term in URL | `inurl:login` |
| `filetype:` | File extension | `filetype:pdf` |
| `intitle:` | Term in `<title>` | `intitle:"confidential report"` |
| `intext:` / `inbody:` | Term in body | `intext:"password reset"` |
| `cache:` | Cached version | `cache:example.com` |
| `link:` | Pages linking to URL | `link:example.com` |
| `related:` | Similar sites | `related:example.com` |
| `info:` | Page summary | `info:example.com` |
| `define:` | Definitions | `define:phishing` |
| `numrange:` | Numbers in range | `site:example.com numrange:1000-2000` |
| `allintext:` | All terms in body | `allintext:admin password reset` |
| `allinurl:` | All terms in URL | `allinurl:admin panel` |
| `allintitle:` | All terms in title | `allintitle:confidential report 2023` |
| `AND` | Require all terms | `site:example.com AND inurl:admin` |
| `OR` | Either term | `"linux" OR "ubuntu" OR "debian"` |
| `NOT` / `-` | Exclude term | `site:bank.com NOT inurl:login` |
| `*` (wildcard) | Any word/char | `user* manual` |
| `..` (range) | Numeric range | `"price" 100..500` |
| `" "` | Exact phrase | `"information security policy"` |

## Google Dorking — High-Value Patterns
| Goal | Dork |
|---|---|
| Find login pages | `site:example.com inurl:login` |
| Find login or admin pages | `site:example.com (inurl:login OR inurl:admin)` |
| Exposed PDFs | `site:example.com filetype:pdf` |
| Exposed Office docs | `site:example.com (filetype:xls OR filetype:docx)` |
| Config files | `site:example.com inurl:config.php` |
| `.conf` / `.cnf` config files | `site:example.com (ext:conf OR ext:cnf)` |
| Backups | `site:example.com inurl:backup` |
| SQL backups | `site:example.com filetype:sql` |

For more: see the **Google Hacking Database (GHDB)** at exploit-db.com/google-hacking-database.

## Use Cases
| Use case | Why search engines work |
|---|---|
| Security assessment | Identify exposed data, login portals, leaked configs |
| Competitive intel | Public information about competitors' tech/services |
| Investigative journalism | Discover hidden connections, financial data |
| Threat intelligence | Track threat actors via leaked artifacts/IOCs |

## Key Takeaways
- Always combine `site:` + one other operator — that's the recon sweet spot.
- Quotes around phrases prevent Google from "fixing" your search.
- Bing and DuckDuckGo index different content than Google — try multiple engines.
- Save time with the GHDB — pre-built dorks for common scenarios.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← [[15-creepy-crawlies]] | [[17-web-archives]] →
<!-- AUTO-LINKS-END -->
