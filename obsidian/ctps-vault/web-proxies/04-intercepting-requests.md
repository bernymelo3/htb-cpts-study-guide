## ID
530

## Module
Using Web Proxies

## Kind
notes

## Title
Section 4 — Intercepting Web Requests

## Description
Covers intercepting and manipulating HTTP requests with Burp Suite and ZAP to bypass front-end validation and perform command injection.

## Tags
burp suite, zap, proxy, intercept, command injection, request manipulation

## Commands
- ip=%3bcat+<FILE>
- ip=%3bls%3b

## What This Section Covers
Web proxies can intercept HTTP requests before they reach the server, allowing testers to modify parameters that front-end JavaScript would otherwise restrict. This is foundational for testing injection flaws — if the back end doesn't validate input independently from the front end, attackers can inject arbitrary commands by editing POST data in transit.

## Methodology
1. Set FoxyProxy to the **Burp (8080)** preset in Firefox.
2. In Burp Suite, confirm **Proxy → Intercept** is toggled **on**.
3. Navigate to `http://STMIP:STMPO/` in the pre-configured browser.
4. Enter any number (e.g. `1`) in the IP field and click **Ping** — Burp captures the POST to `/ping`.
5. Send the intercepted request to **Repeater** with `Ctrl+R`.
6. In Repeater, change the `ip` parameter from `1` to `;cat flag.txt`.
7. URL-encode the payload by highlighting it and pressing `Ctrl+U` — result: `ip=%3bcat+flag.txt`.
8. Click **Send** — the response body contains the flag.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Change the command to read flag.txt | HTB{1n73rc3p73d_1n_7h3_m1ddl3} | Intercept POST `/ping`, change `ip=1` to `ip=%3bcat+flag.txt` in Repeater, send |

## Key Takeaways
- Front-end input validation (JS restricting to numbers) is trivially bypassed — it only exists for UX, not security.
- Always test whether the back end independently validates parameters; if it trusts front-end filtering, injection is likely possible.
- The `;` character terminates the original command and starts a new one — classic OS command injection separator.
- URL-encoding (`Ctrl+U` in Burp) is essential when injecting special characters in POST bodies.
- ZAP's HUD lets you intercept and manipulate requests directly in the browser without switching windows.

## Gotchas
- Firefox may send background requests (update checks, telemetry) that get intercepted before your target request — just click **Forward** until you see the POST to `/ping`.
- Forgetting to URL-encode the payload will cause the server to misparse the POST body and the injection will fail.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-proxies]]  
← [[03-proxy-setup]] | [[05-intercepting-responses]] →
<!-- AUTO-LINKS-END -->
