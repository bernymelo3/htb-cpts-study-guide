## ID
531

## Module
Using Web Proxies

## Kind
notes

## Title
Section 7 — Repeating Requests

## Description
Using Burp Repeater and ZAP Request Editor to resend and modify captured requests for rapid command injection testing without re-intercepting each time.

## Tags
burp, zap, repeater, request-repeating, command-injection, proxy

## Commands
- Ctrl+R to send request to Burp Repeater
- Ctrl+Shift+R to switch to Repeater tab
- ip=127.0.0.1;ls+/
- ip=127.0.0.1;find+/+-name+"flag*"
- ip=127.0.0.1;cat+<FLAG_PATH>

## What This Section Covers
Request repeating lets you resend any previously captured request with modifications, without needing to re-intercept from the browser each time. This dramatically speeds up manual testing like command injection enumeration, where you'd otherwise need 5–6 steps per payload.

## Methodology
1. Submit a normal request through the proxy so it appears in Proxy History (`Proxy > HTTP History` in Burp, bottom History pane in ZAP)
2. Select the request and send it to Repeater with `Ctrl+R` (Burp) or right-click → `Open/Resend with Request Editor` (ZAP)
3. Modify the parameter directly in the Repeater pane (e.g. change `ip=127.0.0.1` to `ip=127.0.0.1;ls+/`)
4. Click `Send` and read the response inline — no browser needed
5. Iterate by editing the same request with new payloads and resending

## Key Takeaways
- Repeater eliminates the intercept→modify→forward→check-browser loop — you edit and send in one place
- Burp keeps both the Original Request and the Edited Request; ZAP only shows the final version
- Request body values must be URL-encoded (spaces as `+`, special chars as `%XX`)
- Right-click → `Change Request Method` in Burp lets you flip between GET/POST without rewriting the request
- ZAP HUD offers `Replay in Console` (see response in HUD) and `Replay in Browser` (render response in the page)
- Proxy History also logs WebSocket connections, useful for advanced testing but out of scope here


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-proxies]]  
← [[06-zap-replacer-notes]] | [[08-encoding-decoding]] →
<!-- AUTO-LINKS-END -->
