# NOTE — Web Archives (Wayback Machine)

## ID
216

## Module
Information Gathering – Web Edition

## Kind
lab

## Title
Section 17 — Web Archives

## Description
The Internet Archive's Wayback Machine stores historical snapshots of websites. For recon: find old subdomains, deprecated endpoints, leaked content that's since been removed, vulnerable versions of pages still archived.

## Tags
wayback-machine, web-archive, internet-archive, passive-recon, historical-data, snapshot

## What This Section Covers
The Wayback Machine (archive.org/web) crawls the public web and stores point-in-time snapshots. For pentesters this is a treasure trove: deleted pages, old subdomains, vulnerable software versions, and forgotten endpoints often live on in archived form long after the live site moves on.

## How It Works
1. **Crawling** — Internet Archive bots browse the public web like search-engine crawlers.
2. **Archiving** — copies are stored linked to a date+time, building a history.
3. **Accessing** — query a URL + date in the calendar UI to view that snapshot.

Crawl frequency varies by site popularity — some sites have hourly snapshots, others years between them. Site owners can request exclusion (not always honored).

## Why Recon Cares
| Use case | Detail |
|---|---|
| **Hidden assets** | Old pages, dirs, files, subdomains may still exist on the live site even if unlinked |
| **Patterns over time** | Compare snapshots to see how the site/tech evolved |
| **OSINT** | Past staff bios, marketing material, technology choices |
| **Stealth** | Browsing the archive doesn't touch the target |

## Workflow
1. Open `https://web.archive.org/`.
2. Enter the target URL.
3. Pick a date from the calendar — sometimes multiple snapshots per day.
4. Browse the archived site as if it were live.
5. Note any URLs/files in the archive that aren't in the current site — try requesting them on the live site (they often still resolve).

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Pen Testing Labs HackTheBox had on 2018-08-08 (integer) | **{see lab — answer redacted in walkthrough}** | Wayback Machine → `https://www.hackthebox.eu/en` → 2018-08-08 → scroll to `[ Pen Testing Labs ]` |
| Q2 — Members HackTheBox had on 2017-06-10 | **{see lab — answer redacted in walkthrough}** | Wayback Machine → `hackthebox.eu/en` → 2017-06-10 → `[ members ]` section |
| Q3 — Domain `facebook.com` redirected to in March 2002 | **{see lab — answer redacted in walkthrough}** | Wayback Machine → `facebook.com` → 2002-03-28 snapshot → follow redirect |
| Q4 — On `paypal.com` October 1999, what could you "beam money to anyone" with? | **{see lab — answer redacted in walkthrough}** | Wayback Machine → `paypal.com` → 1999-10-13 snapshot |
| Q5 — November 1998 Google "Search Engine Prototype" address | **{see lab — answer redacted in walkthrough}** | Wayback Machine → `google.com` → 1998-11-11 → click `Google Search Engine Prototype` link |
| Q6 — On `www.iana.org` March 2000, when was site last updated (footer date)? | **{see lab — answer redacted in walkthrough}** | Wayback Machine → `www.iana.org` → 2000-03-03 → check footer |
| Q7 — `wikipedia.com` on 2003-02-09 — articles in English version (no commas) | **{see lab — answer redacted in walkthrough}** | Wayback Machine → `wikipedia.com` → 2003-02-09 |

## Key Takeaways
- Wayback is fully passive — the target sees nothing.
- Always check archived `/robots.txt` and `/sitemap.xml` — old versions sometimes list paths the current ones hide.
- Old cached endpoints sometimes still work on live sites (deleted from nav, not from disk).
- For automated extraction of archived URLs there are CLI tools (`waybackurls`, `gau`, `cariddi`) — useful when manually browsing the calendar gets tedious.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← [[16-search-engine-discovery]] | [[18-automating-recon]] →
<!-- AUTO-LINKS-END -->
