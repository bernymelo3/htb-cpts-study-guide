## ID
203

## Module
Using Web Proxies

## Kind
notes

## Title
Section 5 — Intercepting Responses

## Description
Teaches how to intercept and modify HTTP responses before they reach the browser, enabling on‑the‑fly HTML changes to bypass client‑side restrictions.

## Tags
intercept, response-modification, burp-suite, zap, hud, client-side

## Commands
- Enable response interception in Burp: Proxy → Proxy Settings → Intercept Server Responses
- In ZAP, press Step to forward the request and automatically intercept the response
- Modify HTML attribute: change `type="number"` to `type="text"` and `maxlength="3"` to `maxlength="100"`
- Use ZAP HUD Show/Enable button to toggle disabled fields without manual interception
- In Burp, automate response modification: Proxy → Proxy Settings → Response Modification Rules → Unhide hidden form fields

## What This Section Covers
Intercepting responses lets you modify the HTML/CSS/JavaScript that the server sends back, before it renders in the browser. This enables you to re‑enable disabled input fields, reveal hidden elements, or alter client‑side validation logic — all without touching the server. It’s a powerful technique for easing manual testing and bypassing client‑side restrictions that aren’t enforced server‑side.

## Methodology
1. In Burp, go to **Proxy → Proxy Settings** and under **Response Interception Rules**, enable **Intercept Server Responses** (tick the checkbox).
2. Turn request interception back on, then in the browser force‑reload the page (Ctrl+Shift+R).
3. In Burp, forward the intercepted request. The response will now be caught and displayed in the Intercept tab.
4. Edit the HTML directly: for example, change `type="number"` to `type="text"` and increase `maxlength` to allow any input.
5. Click **Forward** to release the modified response to the browser.
6. In ZAP, start with a request interception; when the response arrives, ZAP will automatically pause on it (or click **Step** to intercept the response after the request).
7. Make the same HTML changes and click **Continue**.
8. For quick tweaks like enabling disabled fields or revealing hidden elements, use the ZAP HUD’s **Show/Enable** button (the light bulb icon) — no manual interception needed.
9. Optionally, add the **Comments** button in ZAP HUD to highlight HTML comments for extra insight.
10. Burp also offers automatic response modification under **Proxy → Proxy Settings → Response Modification Rules**, e.g., “Unhide hidden form fields”, which applies changes without manual interception.

## Key Takeaways
- Intercepting responses is the reverse of intercepting requests: you alter what the server already sent, not what the client will send.
- It’s especially useful for bypassing client‑side validation that is not re‑checked server‑side (e.g., `maxlength`, `type="number"`).
- ZAP HUD’s “Show/Enable” provides a one‑click shortcut for common HTML modifications.
- Burp’s response modification rules can automate these changes, saving time during repeated tests.
- Always check whether the server re‑enforces the original restriction after you modify the response; if it does, client‑side tampering won’t help.

## Gotchas
- After modifying a response, the browser may cache the altered version; a hard refresh (Ctrl+F5) can revert it if needed.
- Response interception can cause pages to hang if you forget to forward the response — check the Intercept tab.
- Not all disabled fields are client‑only; if the server expects a specific input format, your modified value may be rejected on submission.
- The ZAP HUD’s light‑bulb button only enables/disables fields and reveals hidden elements; it doesn’t allow arbitrary HTML changes like manual interception does.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-proxies]]  
← [[04-intercepting-requests]] | [[06-automatic-modification]] →
<!-- AUTO-LINKS-END -->
