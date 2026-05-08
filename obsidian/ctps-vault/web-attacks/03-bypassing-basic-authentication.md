# NOTE — Web Attacks § 3: Bypassing Basic Authentication

## ID
530

## Module
Web Attacks

## Kind
notes

## Title
Section 3 — Bypassing Basic Authentication

## Description
Demonstrates how insecure web server configurations that only restrict GET/POST methods on protected directories can be bypassed using alternate HTTP verbs like HEAD and OPTIONS.

## Tags
http-verb-tampering, authentication-bypass, head-method, apache, web-server-misconfiguration

## Commands
- curl -i http://<TARGET>:<PORT>/admin/reset.php
- curl -i -X OPTIONS http://<TARGET>:<PORT>/
- curl -i -X HEAD http://<TARGET>:<PORT>/admin/reset.php
- curl -i -X POST http://<TARGET>:<PORT>/admin/reset.php

## What This Section Covers
HTTP Verb Tampering exploits web server configurations that only enforce authentication on specific HTTP methods (typically GET and POST), leaving other methods like HEAD, OPTIONS, and PATCH unrestricted. By sending a request with an uncovered verb, attackers can execute server-side logic behind protected endpoints without providing credentials.

## Methodology
1. Browse the target app and identify restricted functionality (e.g. clicking Reset → `401 Unauthorized` on `/admin/reset.php`)
2. Confirm the entire `/admin/` directory requires auth by visiting it directly
3. Check accepted HTTP methods with `curl -i -X OPTIONS http://<TARGET>:<PORT>/` — look for the `Allow:` header
4. Send the request using an alternate verb: `curl -i -X HEAD http://<TARGET>:<PORT>/admin/reset.php`
5. HEAD returns `200 OK` with an empty body — the server-side code executes but the auth check is skipped
6. Revisit the root page to confirm the action completed (files deleted, flag displayed)

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Access reset.php and delete all files | HTB{4lw4y5_c0v3r_4ll_v3rb5} | Sent HEAD request to `/admin/reset.php` bypassing basic auth, then refreshed root page |

## Key Takeaways
- Server auth configs (`.htaccess` `<Limit GET POST>`) that only list specific methods leave every other verb wide open — always use `<LimitExcept>` instead
- HEAD is functionally identical to GET (server executes the same logic) but returns no body — perfect for triggering actions silently
- OPTIONS reveals the full list of accepted methods and is itself often unrestricted
- Automated scanners catch insecure server configs but often miss verb tampering caused by insecure application-level code
- Always test multiple HTTP methods (HEAD, OPTIONS, PATCH, PUT, DELETE) when you hit an auth wall

## Gotchas
- HEAD returns no response body, so you won't see output — verify the action took effect by checking the app state separately
- POST alone may still be restricted; the bypass depends on which specific verbs the config omits
- Burp Suite's "Change Request Method" only toggles between GET and POST — for HEAD/OPTIONS/PATCH you need to manually edit the method in the raw request
