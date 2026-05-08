## ID
530

## Module
Command Injections

## Kind
notes

## Title
Section 5 — Identifying Filters

## Description
Covers how to detect which injection operators and characters are blacklisted by a web application's filter/WAF by testing each one individually.

## Tags
command-injection, filter-evasion, blacklist, waf, burp-suite, operators

## Commands
- `127.0.0.1%0a` — newline operator (not blacklisted)
- `127.0.0.1%26` — URL-encoded `&` (blacklisted)
- `127.0.0.1%7c` — URL-encoded `|` (blacklisted)
- `127.0.0.1%3b` — URL-encoded `;` (blacklisted)

## What This Section Covers
Web applications may use character blacklists or WAFs to block injection operators like `;`, `&`, and `|`. This section teaches how to systematically identify which characters are blocked by reducing payloads to one character at a time and testing each operator individually through Burp Suite.

## Methodology
1. Submit a normal request (`127.0.0.1`) and confirm it works — this is your baseline.
2. Intercept the POST request in Burp Suite and send it to Repeater.
3. Append a single operator at a time to the `ip` parameter to isolate which ones trigger "Invalid input":
   - `ip=127.0.0.1%3b` (`;`) → blocked
   - `ip=127.0.0.1%26` (`&`) → blocked
   - `ip=127.0.0.1%7c` (`|`) → blocked
   - `ip=127.0.0.1%0a` (newline `\n`) → **not blocked**
4. Once you find an unblocked operator, you have your injection vector for further exploitation.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Which of (new-line, &, \|) is not blacklisted? | %0a | Tested each URL-encoded operator individually in Burp Repeater; `%0a` returned normal output instead of "Invalid input" |

## Key Takeaways
- Always test operators **one at a time** — isolate the exact character that triggers the filter before trying full payloads.
- Newline (`%0a`) is a valid command separator in Linux shells but is frequently overlooked in blacklists.
- "Invalid input" displayed in the page output = app-level filter. A redirect to a separate error page with your IP/request details = WAF.
- URL-encode operators in Burp Repeater since the request is a POST with URL-encoded body (`%0a`, `%26`, `%7c`, `%3b`).
- PHP `strpos()` blacklist approach (checking each character against a list) is a common but incomplete mitigation — it only catches what the developer thought to include.

## Gotchas
- Don't forget to URL-encode operators in the POST body — sending a raw `&` will be interpreted as a parameter separator, not as part of the `ip` value.
- A blacklist that blocks `;`, `&`, and `|` may still miss `\n`, `&&` vs `&`, or other lesser-known separators like `%0d%0a` (CRLF).
