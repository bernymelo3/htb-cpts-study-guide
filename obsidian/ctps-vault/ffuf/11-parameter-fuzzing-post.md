# THEORY — Parameter Fuzzing (POST)

## ID
11

## Module
Ffuf

## Kind
theory

## Title
Section 11 — Parameter Fuzzing — POST

## Description
POST data goes in the request body, not the URL. Use `-X POST -d 'FUZZ=key'` plus the right `Content-Type` to fuzz body parameter names.

## Tags
ffuf, parameter-fuzzing, post, body, content-type

## Commands
- `ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://admin.academy.htb:PORT/admin/admin.php -X POST -d 'FUZZ=key' -H 'Content-Type: application/x-www-form-urlencoded' -fs xxx`
- `curl http://admin.academy.htb:PORT/admin/admin.php -X POST -d 'id=key' -H 'Content-Type: application/x-www-form-urlencoded'`

## TL;DR — What's Important
- POST params **do not appear in the URL** — they're in the body.
- ffuf flags: `-X POST` to switch method, `-d 'FUZZ=key'` to set body, `-H 'Content-Type: application/x-www-form-urlencoded'` for PHP.
- PHP only auto-parses `application/x-www-form-urlencoded`. Without that header, `$_POST` will be empty and every fuzz looks like a miss.
- Filtering still applies — the baseline (size from a wrong param) needs to be filtered with `-fs`.

## Why fuzz both GET and POST
- Apps frequently accept the same parameter via either method.
- Some parameters are **POST-only** (defensive design) — GET fuzzing won't see them at all.
- Both pass should be tried before concluding a page has no fuzzable input.

## GET vs POST Fuzzing Cheatsheet
| Aspect | GET | POST |
|--------|-----|------|
| FUZZ goes in | URL query string `?FUZZ=key` | Body `-d 'FUZZ=key'` |
| Method flag | (default GET) | `-X POST` |
| Content-Type | n/a | `-H 'Content-Type: application/x-www-form-urlencoded'` |
| Visible in URL? | Yes | No |
| Logged by default proxies? | Yes (URL) | Body — depends on logger |

## Key Takeaways
- A POST-only parameter is a strong hint that the developer wanted to **hide** it from casual log inspection — usually means it's interesting.
- The `Invalid id!` style response (different from "no access") is itself a signal: it confirms the parameter is wired and being parsed.
- After confirming the parameter, fuzz its **value** to find the magic input (section 12).

## Gotchas
- Drop the `Content-Type` header on a PHP target and you'll get nothing back — every request will look like a miss.
- Some frameworks (Express, etc.) accept JSON bodies — `-d '{"FUZZ":"key"}' -H 'Content-Type: application/json'`. Try both encodings.
- Always re-baseline `-fs` per endpoint — POST baselines differ from GET baselines.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ffuf]]
← [[10-parameter-fuzzing-get]] | [[12-value-fuzzing]] →
<!-- AUTO-LINKS-END -->
