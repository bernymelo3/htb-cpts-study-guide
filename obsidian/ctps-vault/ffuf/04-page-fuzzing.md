# LAB — Page Fuzzing (Extension + File)

## ID
4

## Module
Ffuf

## Kind
lab

## Title
Section 4 — Page Fuzzing

## Description
Two-step fuzz inside a directory: first identify the file extension (e.g. `.php`), then enumerate filenames with that extension to find hidden pages.

## Tags
ffuf, page-fuzzing, extension-fuzzing, lab, php

## Commands
- `ffuf -w /opt/useful/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ -u http://SERVER_IP:PORT/blog/indexFUZZ`
- `ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://SERVER_IP:PORT/blog/FUZZ.php`

## What This Section Covers
Once you've found a directory, the next question is "what file types live here?" Look at the `Server:` HTTP header for a hint (Apache → likely PHP, IIS → ASP/ASPX), but the reliable way is to fuzz extensions against a known-existing filename like `index`. After confirming the extension, fuzz filenames with that extension fixed.

## Methodology
1. **Identify extension** — fuzz against a likely filename anchor (`index`):
   `ffuf -w web-extensions.txt:FUZZ -u http://target/dir/indexFUZZ`
2. **Note `Status: 200` hits.** `200 size 0` on `.php` confirms PHP works (just empty body).
3. **Switch to filename fuzzing** with the extension nailed down:
   `ffuf -w directory-list-2.3-small.txt:FUZZ -u http://target/dir/FUZZ.php`
4. **Visit hits with non-zero size** — those are real pages with content.

## Why fuzz `index<ext>` not just any file
- `index.*` exists on virtually every webserver (default page).
- The wordlist already includes the leading dot, so write `indexFUZZ` (not `index.FUZZ`).

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Flag from fuzzing `/blog` for pages | **HTB{bru73_f0r_c0mm0n_p455w0rd5}** | `ffuf -s -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u 'http://STMIP:STMPO/blog/FUZZ.php'` returned `index` and `home`. Visiting `/blog/home.php` shows the flag |

## Key Takeaways
- Two-stage fuzz (extension first, filename second) is faster than fuzzing both at once with two `FUZZ`/`FUZ2Z` keywords.
- `200, Size: 0` on a fuzzed extension means "extension is parsed by the server" — proof the language is enabled.
- A file responding `403` Forbidden also confirms the extension is parsed (server recognises and refuses it).

## Gotchas
- The `web-extensions.txt` wordlist already includes the dot — don't add a second one or you'll get `index..php`.
- Some servers serve PHP source via `.phps` (PHP source view). Treat `.phps` 200 hits as a possible info-disclosure jackpot.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ffuf]]
← [[03-directory-fuzzing]] | [[05-recursive-fuzzing]] →
<!-- AUTO-LINKS-END -->
