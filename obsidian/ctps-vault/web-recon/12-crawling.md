# NOTE — Crawling

## ID
211

## Module
Information Gathering – Web Edition

## Kind
theory

## Title
Section 12 — Crawling

## Description
Crawling vs fuzzing: a crawler systematically follows existing links from a seed URL, while fuzzing guesses paths. Covers breadth-first vs depth-first strategies and the categories of data worth extracting (links, comments, metadata, sensitive files).

## Tags
crawling, spidering, breadth-first, depth-first, links, comments, metadata, sensitive-files

## TL;DR — What's Important
- **Crawling = following existing links**. **Fuzzing = guessing paths**. Two different techniques.
- Crawler starts with a seed URL → fetches → parses → enqueues every link → repeats.
- Two strategies: breadth-first (wide overview) vs depth-first (deep into one path).
- Useful extractions: internal/external links, comments (devs leave gold there), metadata, sensitive files (`.bak`, `.old`, `web.config`, `error_log`).

## Concept Overview
A web crawler is an automated bot that systematically browses links to discover and catalogue resources. Search engines crawl to index; pentesters crawl to map. The key insight: a crawler can only find what's *linked* somewhere — anything orphaned needs fuzzing instead.

## How a Crawler Works
1. Start with a seed URL.
2. Fetch the page.
3. Parse HTML/CSS/JS for every URL reference.
4. Add unseen URLs to the queue.
5. Pop the next URL, repeat.

## Crawling vs Fuzzing
| | Crawling | Fuzzing |
|---|---|---|
| Discovery method | Follows existing links | Guesses paths from wordlist |
| Finds | Linked resources | Hidden / unlinked resources |
| Tools | Burp Spider, ZAP, Scrapy, Nutch | ffuf, gobuster, feroxbuster |
| When to use | First-pass mapping | After crawling, to find what's hidden |

## Breadth-First vs Depth-First
| Strategy | Behaviour | Best for |
|---|---|---|
| **Breadth-first** | Explore all links on current page first, then go one level deeper | Quick overview of site structure |
| **Depth-first** | Follow one path to its leaf, then backtrack | Reaching deep nested resources |

## What to Extract
| Data | Why it matters |
|---|---|
| **Internal links** | Map site structure, find orphan pages reachable only from one obscure place |
| **External links** | 3rd-party services in use (analytics, CDNs, SaaS) |
| **HTML comments** | Devs leave dev-mode notes, TODOs, credentials, hidden URLs (`<!-- TODO: change API key to ... -->`) |
| **Metadata** | Page titles, descriptions, author names, dates — context clues |
| **Sensitive files** | `.bak`, `.old`, `.swp`, `web.config`, `settings.php`, `error_log`, `access_log` — accidental exposure of secrets |

## The Importance of Context
A single comment about a "file server" is mundane. Combine it with a discovered `/files/` directory that has directory browsing enabled, and you've got a high-priority finding. Crawler output is most valuable when correlated, not read line-by-line.

## Key Takeaways
- Crawling and fuzzing are complementary — always do both.
- Comments and metadata are often the highest-signal byproducts of a crawl.
- Always look for sensitive file extensions in the link list (`.bak`, `.old`, `.git`, `.env`).
- Holistic analysis > line-by-line — connect findings across pages.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← [[11-fingerprinting]] | [[13-robots-txt]] →
<!-- AUTO-LINKS-END -->
