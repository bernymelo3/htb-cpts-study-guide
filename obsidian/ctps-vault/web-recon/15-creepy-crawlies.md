# NOTE — Creepy Crawlies (Scrapy + ReconSpider)

## ID
214

## Module
Information Gathering – Web Edition

## Kind
lab

## Title
Section 15 — Creepy Crawlies

## Description
Tour of crawler tooling (Burp Spider, ZAP, Scrapy, Apache Nutch) plus a hands-on with `ReconSpider.py` — the HTB-provided Scrapy spider that dumps emails, links, JS files, comments, etc. into `results.json`.

## Tags
crawling, scrapy, reconspider, burp-spider, zap, apache-nutch, results-json, jq

## Commands
- `pip3 install scrapy`
- `wget -O ReconSpider.zip https://academy.hackthebox.com/storage/modules/144/ReconSpider.v1.2.zip`
- `unzip ReconSpider.zip`
- `python3 ReconSpider.py http://inlanefreight.com`
- `cat results.json | jq '.comments'`
- `cat results.json | jq '.emails'`
- `cat results.json | jq '.links'`

## What This Section Covers
The major crawler tools and how to drive `ReconSpider.py` (the HTB-provided custom Scrapy spider) against a target. Output is a structured `results.json` with categorised extractions — emails, links, external files, JS, images, audio, video, form fields, comments.

## Tool Tour
| Tool | Best for |
|---|---|
| **Burp Spider** | Active spider inside Burp Suite — tightly integrated with the proxy/scanner |
| **OWASP ZAP** | Free open-source alternative to Burp; spider + active scanner |
| **Scrapy** (Python) | Custom crawlers — full control, programmatic data processing |
| **Apache Nutch** | Massive-scale crawls (whole web or large domain sets); Java-based |

## ReconSpider Workflow
1. **Install Scrapy**: `pip3 install scrapy` (`--break-system-packages` on newer Debian/Ubuntu/Pwnbox).
2. **Download + extract**:
   ```bash
   wget -O ReconSpider.zip https://academy.hackthebox.com/storage/modules/144/ReconSpider.v1.2.zip
   unzip ReconSpider.zip
   ```
3. **Run against target**:
   ```bash
   python3 ReconSpider.py http://inlanefreight.com
   ```
4. **Examine `results.json`** with `jq` — focus on the high-signal keys first (`comments`, `emails`).

## results.json Schema
| Key | Contents |
|---|---|
| `emails` | Email addresses harvested from the site |
| `links` | All discovered URLs |
| `external_files` | Off-site files linked (PDFs, etc.) |
| `js_files` | All `<script src="...">` URLs |
| `form_fields` | Form input names |
| `images` | All `<img src="...">` URLs |
| `videos` | Video file URLs |
| `audio` | Audio file URLs |
| `comments` | HTML comments — usually the highest-value field |

## Why `comments` Is Goldmine #1
Devs leave TODOs, deprecated URLs, internal email addresses, and even credentials in HTML comments. A typical golden find:
```html
<!-- TO-DO: change the location of future reports to files.inlanefreight.com -->
<!-- change Jeremy's email to jeremy-ceo@inlanefreight.com -->
<!-- Remember to change the API key to <new key> -->
```
Pull them with:
```bash
cat results.json | jq '.comments'
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — After spidering `inlanefreight.com`, identify where future reports will be stored | **{see lab — answer redacted in walkthrough}** | `python3 ReconSpider.py http://inlanefreight.com` then `cat results.json \| jq '.comments'` — look for the `<!-- TO-DO: change the location of future reports to ... -->` comment |

## Key Takeaways
- For one-off crawls, ReconSpider + jq is faster than Burp/ZAP for extracting structured data.
- `jq '.comments'` is the single most useful query against the output — start there.
- Use `pip3 install scrapy --break-system-packages` on modern Debian-based systems (PEP 668).
- Always inspect every `jq` key — `emails`, `js_files`, and `external_files` often reveal third-party services and forgotten endpoints.

## Gotchas
- ReconSpider only follows HTTP — for JS-heavy SPAs, you'll miss links rendered client-side. Use a headless-browser crawler (Playwright/Puppeteer) for those.
- The crawler hits robots.txt by default in many configs — vhost-only or non-public sub-paths may be skipped.
- Be polite — set rate limits in production. The lab targets are forgiving; real ones aren't.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← [[14-well-known-uris]] | [[16-search-engine-discovery]] →
<!-- AUTO-LINKS-END -->
