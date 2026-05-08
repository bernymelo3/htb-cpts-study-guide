# NOTE — Web Attacks § 4: Bypassing Security Filters

## ID
531

## Module
Web Attacks

## Kind
notes

## Title
Section 4 — Bypassing Security Filters

## Description
Demonstrates how insecure coding that only validates input from one HTTP method (e.g. `$_GET`) can be bypassed by switching to another method, enabling command injection through HTTP Verb Tampering.

## Tags
http-verb-tampering, command-injection, security-filter-bypass, insecure-coding, php

## Commands
- curl -s "http://<TARGET>:<PORT>/" -G --data-urlencode "filename=file; cp /flag.txt ./"
- curl -s "http://<TARGET>:<PORT>/" -d "filename=file; cp /flag.txt ./"
- curl -s http://<TARGET>:<PORT>/flag.txt

## What This Section Covers
When developers use method-specific variables like `$_GET['param']` in security filters but method-agnostic variables like `$_REQUEST['param']` in the actual application logic, attackers can bypass the filter entirely by switching HTTP methods. This is the more common and harder-to-detect form of HTTP Verb Tampering — caused by insecure code rather than server misconfiguration.

## Methodology
1. Identify the filter — submit a filename with special characters (e.g. `test;`) and observe the `Malicious Request Denied!` response
2. Intercept the request in Burp Suite and note it uses GET
3. Right-click → **Change Request Method** to switch to POST
4. Resend with a benign filename to confirm the filter is bypassed (no denial message)
5. Now inject the payload: set filename to `file; cp /flag.txt ./` and send as POST
6. Turn off Burp interception, browse back to the root page, and click `flag.txt` to read the flag

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Bypass filter with `file; cp /flag.txt ./` | HTB{b3_v3rb_c0n51573n7} | Changed GET to POST in Burp, injected command, then accessed `/flag.txt` |

## Key Takeaways
- Insecure coding verb tampering is more common than server misconfiguration verb tampering, and scanners typically miss it
- The root cause is a mismatch: filter checks `$_GET` but the app processes `$_REQUEST` (which accepts both GET and POST)
- The fix is to use `$_REQUEST` consistently in both the filter and the logic, or explicitly validate all input regardless of method
- This pattern extends beyond PHP — any framework where input source and filter source diverge is vulnerable
- Always test both GET→POST and POST→GET swaps when you hit an input filter wall

## Gotchas
- After forwarding the modified request in Burp, turn interception OFF before browsing back — otherwise you'll have to manually forward every subsequent request
- The injected `cp` command copies the flag to the web root, but the file only appears after refreshing the page
- Burp's "Change Request Method" only toggles GET↔POST — for other verbs you still need to edit manually
