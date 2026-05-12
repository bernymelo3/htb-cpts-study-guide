# NOTE — robots.txt

## ID
212

## Module
Information Gathering – Web Edition

## Kind
theory

## Title
Section 13 — robots.txt

## Description
`robots.txt` lives at the site root and tells crawlers what NOT to index. From a recon perspective: every `Disallow:` line is a finger pointing at something the site owner wants hidden — admin panels, backups, internal docs.

## Tags
robots-txt, crawling, recon, hidden-paths, disallow, allow, sitemap, honeypot

## TL;DR — What's Important
- `robots.txt` = `/robots.txt` at site root. Plain text. Etiquette guide for bots.
- **Not enforced** — it's a request, not a restriction. Malicious bots ignore it.
- For recon: `Disallow:` lines are gifts. They tell you exactly where the interesting stuff lives.
- Common directives: `User-agent`, `Disallow`, `Allow`, `Crawl-delay`, `Sitemap`.

## Concept Overview
`robots.txt` follows the Robots Exclusion Standard. It's how website owners ask well-behaved crawlers (Googlebot, Bingbot, etc.) to stay out of certain paths. Pentesters care because every "stay out" entry advertises exactly where to look first.

## File Structure
- Lives at the **site root**: `https://example.com/robots.txt`.
- One or more "records," separated by blank lines.
- Each record = `User-agent` line + one or more directives.

## Directive Reference
| Directive | Purpose | Example |
|---|---|---|
| `User-agent` | Which bot the rules apply to (`*` = all) | `User-agent: *` |
| `Disallow` | Path the bot should not crawl | `Disallow: /admin/` |
| `Allow` | Explicit permit (overrides `Disallow`) | `Allow: /public/` |
| `Crawl-delay` | Seconds between requests | `Crawl-delay: 10` |
| `Sitemap` | URL of an XML sitemap | `Sitemap: https://example.com/sitemap.xml` |

## Example
```
User-agent: *
Disallow: /admin/
Disallow: /private/
Allow: /public/

User-agent: Googlebot
Crawl-delay: 10

Sitemap: https://www.example.com/sitemap.xml
```
Tells you: there's an admin panel at `/admin/`, private content at `/private/`, and an XML sitemap to slurp.

## Why Recon Cares
| Use | Detail |
|---|---|
| **Hidden directories** | Disallowed paths often = sensitive content |
| **Site structure mapping** | Reveals sections not in the main nav |
| **Honeypot detection** | Some sites bait `Disallow: /honeypot/` to detect bots that ignore robots.txt — a hint about the org's security maturity |
| **Sitemap.xml pointer** | One file lists every indexable URL — saves crawling |

## Why Respect robots.txt (defender's view)
- Avoids overloading servers with crawler traffic.
- Some content is truly private and shouldn't be indexed.
- Ignoring it can violate ToS or expose you legally if accessing copyrighted/private data.

## Recon Tradecraft
1. **Always check `/robots.txt` first** — it's free intel.
2. **Diff `Disallow:` paths against your fuzzing wordlist** — add them to the list.
3. **Always check `sitemap.xml`** if listed — it's the structured site map.
4. **Don't blindly request every Disallow path on a live target** in a noisy way — those paths are often instrumented for "anyone hitting this is suspicious."

## Key Takeaways
- `Disallow:` ≠ inaccessible. It's a request to crawlers, nothing more.
- Always pull `robots.txt` and `sitemap.xml` as the very first crawl step.
- Be aware of honeypot entries — first hit may flag your IP.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← [[12-crawling]] | [[14-well-known-uris]] →
<!-- AUTO-LINKS-END -->
