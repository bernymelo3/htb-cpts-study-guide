# NOTE — Stored XSS

## ID
600

## Module
Cross-Site Scripting (XSS)

## Kind
notes

## Title
Section 2 — Stored XSS

## Description
Covers Stored (Persistent) XSS — the most critical XSS type — where injected payloads are saved in the back-end database and execute for every user who visits the page.

## Tags
xss, stored-xss, persistent-xss, javascript, cookie-stealing

## Commands
- `<script>alert(window.origin)</script>`
- `<script>alert(document.cookie)</script>`
- `<script>print()</script>`
- `<plaintext>`

## What This Section Covers
Stored XSS occurs when user input is saved to the back-end database and rendered unsanitized on page load, meaning the payload fires for every visitor — not just the attacker. This makes it the most dangerous XSS type. The section demonstrates injecting payloads into a To-Do List app and verifying persistence by refreshing the page.

## Methodology
1. Identify an input field that stores and reflects user data (e.g., a To-Do List item, comment box, profile field).
2. Test with a basic XSS payload: `<script>alert(window.origin)</script>` — if an alert pops showing the page URL, the input is vulnerable.
3. Confirm persistence by refreshing the page — if the alert fires again, it's Stored XSS.
4. Verify in page source (`CTRL+U`) that the payload is embedded in the DOM unsanitized.
5. To exfiltrate cookies, swap the payload: `<script>alert(document.cookie)</script>`.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 | Submit the cookie value from the alert | Injected `<script>alert(document.cookie)</script>` into the To-Do List input field; the alert displayed the cookie containing the flag |

## Key Takeaways
- Stored XSS is the most critical type because it affects **every visitor**, not just the attacker — it's a one-to-many attack.
- Always check `window.origin` in the alert to confirm which domain/iframe the XSS actually executes on — cross-domain iframes may contain the vuln without affecting the main app.
- If `alert()` is blocked by the browser, use `<script>print()</script>` (triggers print dialog) or `<plaintext>` (stops HTML rendering) as alternative confirmation payloads.
- `document.cookie` gives you the cookies accessible to JavaScript — cookies with `HttpOnly` flag won't appear here.
- Persistence check: refresh the page. If the payload fires again, it's stored in the DB, not just reflected.

## Gotchas
- Some browsers block `alert()` in certain contexts (sandboxed iframes, etc.) — doesn't mean XSS failed, just use an alternative payload like `print()`.
- The reset button on the lab clears stored payloads from the DB — use it if you need a clean slate.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|xss]]  
← [[01-intro]] | [[03-reflected-xss]] →
<!-- AUTO-LINKS-END -->
