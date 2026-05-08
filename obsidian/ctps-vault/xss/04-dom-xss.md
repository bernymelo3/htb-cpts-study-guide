# NOTE — DOM XSS

## ID
602

## Module
Cross-Site Scripting (XSS)

## Kind
notes

## Title
Section 4 — DOM XSS

## Description
Covers DOM-based XSS — a non-persistent client-side-only vulnerability where JavaScript writes unsanitized user input to the DOM without any back-end server interaction.

## Tags
xss, dom-xss, innerhtml, client-side, source-sink, onerror

## Commands
- `<img src="" onerror=alert(window.origin)>`
- `<img src="" onerror=alert(document.cookie)>`
- `http://<TARGET>:<PORT>/#task=<img src="" onerror=alert(document.cookie)>`

## What This Section Covers
DOM XSS is entirely client-side — no HTTP request is sent to the back-end. JavaScript reads user input (the **Source**, e.g., a URL parameter after `#`) and writes it to the page via a **Sink** function (e.g., `innerHTML`). If the Sink doesn't sanitize input, it's vulnerable. Unlike Reflected XSS, DOM XSS never touches the server, so it won't appear in server logs or WAF alerts.

## Methodology
1. Enter test input and check DevTools **Network** tab — if no HTTP requests are made, processing is client-side only.
2. Check the URL — a `#` fragment (e.g., `#task=test`) confirms client-side parameter handling (fragments are never sent to the server).
3. View page source (`CTRL+U`) — your input won't appear because the DOM was modified after page load. Use the **Inspector** (`CTRL+Shift+C`) to see the rendered DOM.
4. Read the JavaScript source (e.g., `script.js`) to identify the **Source** (where input is read) and the **Sink** (where it's written to the DOM).
5. Common vulnerable Sinks: `document.write()`, `innerHTML`, `outerHTML`, jQuery's `append()`, `after()`, `add()`.
6. `<script>` tags won't work inside `innerHTML` — use event-based payloads instead: `<img src="" onerror=alert(document.cookie)>`.
7. Craft the URL with the payload in the fragment: `http://<TARGET>:<PORT>/#task=<img src="" onerror=alert(document.cookie)>`.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 | Submit the cookie value from the alert | Navigated to `http://<TARGET>:<PORT>/#task=<img src="" onerror=alert(document.cookie)>` — alert displayed the cookie/flag |

## Key Takeaways
- DOM XSS is **fully client-side** — zero back-end interaction. The `#` fragment never leaves the browser, so server-side WAFs and logging are blind to it.
- **Source & Sink model**: Source = where JS reads input (URL param, input field). Sink = where JS writes it to the DOM. If the Sink doesn't sanitize, you have DOM XSS.
- `innerHTML` blocks `<script>` tags as a security feature — bypass with event handlers like `onerror`, `onload`, `onmouseover` on HTML elements.
- `<img src="" onerror=...>` works because the empty `src` always fails, guaranteeing the `onerror` handler fires.
- Page source (`CTRL+U`) won't show DOM XSS payloads — they only exist in the rendered DOM after JavaScript executes. Always use the Inspector.

## Gotchas
- Don't waste time trying `<script>` payloads inside `innerHTML` — they're silently ignored. Jump straight to event-handler payloads.
- The `#` fragment is key: if you see input after `#` in the URL, you're likely dealing with DOM XSS, not Reflected.
- Some browsers may URL-encode the payload in the address bar — if the payload breaks, try manually typing it or using the input field directly.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|xss]]  
← [[03-reflected-xss]] | [[05-xss-discovery]] →
<!-- AUTO-LINKS-END -->
