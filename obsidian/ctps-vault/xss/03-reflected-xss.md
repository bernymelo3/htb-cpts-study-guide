# NOTE — Reflected XSS

## ID
601

## Module
Cross-Site Scripting (XSS)

## Kind
notes

## Title
Section 3 — Reflected XSS

## Description
Covers Reflected (Non-Persistent) XSS — where user input is returned by the back-end in the HTTP response without sanitization, but is not stored in the database.

## Tags
xss, reflected-xss, non-persistent, get-parameter, url-payload

## Commands
- `<script>alert(window.origin)</script>`
- `<script>alert(document.cookie)</script>`
- `http://<TARGET>:<PORT>/index.php?task=<script>alert(document.cookie)</script>`

## What This Section Covers
Reflected XSS happens when user input is echoed back in the server's response (e.g., error messages, confirmation text) without sanitization, but is not saved to the database. The payload only fires once per request — refreshing the page without the payload in the URL won't trigger it again. Attack delivery requires tricking the victim into clicking a crafted URL.

## Methodology
1. Identify input fields where your text is reflected back in the page (error messages, search results, confirmation text).
2. Inject `<script>alert(window.origin)</script>` — if it fires, the reflected input is not sanitized.
3. Confirm it's Reflected (not Stored) by refreshing the page — the alert should **not** fire again.
4. Open DevTools (`CTRL+Shift+I`) → Network tab to check if the request is GET or POST. GET means the payload travels in the URL.
5. Copy the full URL with the payload from the address bar — this is your weaponized link: `http://<TARGET>:<PORT>/index.php?task=<script>alert(document.cookie)</script>`.
6. Send the crafted URL to the victim — payload executes when they visit it.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 | Submit the cookie value from the alert | Navigated to `http://<TARGET>:<PORT>/index.php?task=<script>alert(document.cookie)</script>` — alert displayed the cookie/flag |

## Key Takeaways
- Reflected XSS is **non-persistent** — it only fires when the victim loads the crafted request. No database storage involved.
- Attack requires **social engineering** — you must get the victim to click your crafted URL (phishing email, forum post, etc.).
- Check the HTTP method via DevTools Network tab: GET requests carry the payload in the URL (easy to weaponize), POST requests require more effort (e.g., a form on an attacker-controlled page that auto-submits).
- The payload appears in the page source wrapped in the error message, confirming the server reflects input directly into the DOM.
- Reflected XSS affects **only the targeted user**, unlike Stored XSS which hits everyone.

## Gotchas
- The `<script>` tag content doesn't render visually — so the error message shows empty quotes `''` where the payload is. Don't mistake this for a failed injection; check page source to confirm.
- URL-encoded characters may break the payload if you copy-paste from certain tools — test in the browser directly first.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|xss]]  
← [[02-stored-xss]] | [[04-dom-xss]] →
<!-- AUTO-LINKS-END -->
