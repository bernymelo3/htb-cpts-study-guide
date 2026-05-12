# LAB — Parameter Fuzzing (GET)

## ID
10

## Module
Ffuf

## Kind
lab

## Title
Section 10 — Parameter Fuzzing — GET

## Description
Place FUZZ where the query-string parameter name goes (`?FUZZ=key`) to enumerate hidden / undocumented GET parameters that the page accepts.

## Tags
ffuf, parameter-fuzzing, get, query-string, lab

## Commands
- `ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://admin.academy.htb:PORT/admin/admin.php?FUZZ=key -fs xxx`

## What This Section Covers
Apps often accept undocumented parameters (debug flags, legacy names, internal-only flags). These tend to be **less hardened** because no one is testing them. Fuzzing parameters means swapping FUZZ into the parameter name and looking for response divergence from the baseline.

## Methodology
1. Identify a target endpoint that requires *some* input (e.g. an admin page returning "You don't have access").
2. Append `?FUZZ=key` to the URL — `key` is just a placeholder value.
3. Use `burp-parameter-names.txt` as the wordlist.
4. Run once unfiltered, note the baseline `Size:`.
5. Re-run with `-fs <baseline-size>` to surface only divergent hits.
6. Manually request the URL with the discovered parameter and inspect the page change.

## Why this matters beyond CTFs
- Hidden parameters surface attack surface that automated scanners miss.
- Found a `debug=1` parameter? It may print stack traces.
- Found `admin=true`? Try it — auth-bypass via parameter is a real bug class.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Accepted GET parameter on `/admin/admin.php` | **user** | `ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u 'http://admin.academy.htb:STMPO/admin/admin.php?FUZZ=key' -fs 798` returned `user` |

## Key Takeaways
- Even a "deprecated" parameter response is a win — it confirms a parameter name and may still be partially wired.
- Parameter discovery is **always** combined with filter calibration (`-fs` or `-ac`) because every wrong parameter returns the same baseline.
- After finding the param, fuzz **values** for it (next section).

## Gotchas
- Some apps validate the parameter case (`User` vs `user`) — try both casings if the obvious form misses.
- If the page accepts both GET and POST, sometimes only one method is wired. If GET fuzzing yields nothing, switch to POST.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ffuf]]
← [[09-filtering-results]] | [[11-parameter-fuzzing-post]] →
<!-- AUTO-LINKS-END -->
