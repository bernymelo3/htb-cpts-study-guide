# THEORY — Web Fuzzing

## ID
2

## Module
Ffuf

## Kind
theory

## Title
Section 2 — Web Fuzzing

## Description
Defines fuzzing as a testing technique and introduces the SecLists wordlists used for the rest of the module.

## Tags
ffuf, theory, seclists, wordlists, web-fuzzing

## TL;DR — What's Important
- **Fuzzing = throw varied input at an interface and read the reaction.** Same idea as SQLi probes, BoF length-walks, or directory brute-force.
- 200 = page exists. 404 = doesn't. ffuf reads the response code to keep/discard.
- **SecLists** is the wordlist canon. Already on Pwnbox at `/opt/useful/seclists/`.
- Default directory wordlist for this module: `Discovery/Web-Content/directory-list-2.3-small.txt`.

## Concept Overview
Web fuzzing brute-forces a wordlist of common paths/files/params against a target and uses HTTP codes to filter what exists. Quality matters — a list of 90k common directory names finds ~90% of pages on most sites. Custom or randomly-named pages still slip through; that's why parameter and value fuzzing exist as the next layer.

## Wordlists — Where They Live
| Path | Use |
|------|-----|
| `/opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt` | Directories + page names |
| `/opt/useful/seclists/Discovery/Web-Content/web-extensions.txt` | File extensions (`.php`, `.asp`, ...) |
| `/opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt` | GET/POST parameter names |
| `/opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt` | Sub-domains / vhosts |
| `/opt/useful/seclists/Usernames/Names/names.txt` | Username values |

## Key Takeaways
- The "small" `directory-list-2.3` already covers ~87 k entries and resolves in <10 s with default threads (40).
- The wordlist starts with a copyright comment block — pass `-ic` to ffuf to skip those lines and avoid clutter.
- Locate the file fast with `locate directory-list-2.3-small.txt`.

## Gotchas
- A 200 OK does not mean the page is meaningful — `Size: 0` usually means an empty/redirect page.
- Wordlist quality is everything. Don't reach for the largest list first; start small, escalate if you find nothing.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ffuf]]
← [[01-introduction]] | [[03-directory-fuzzing]] →
<!-- AUTO-LINKS-END -->
