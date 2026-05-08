## ID
202

## Module
Using Web Proxies

## Kind
notes

## Title
Section 3 — Proxy Setup

## Description
Explains how to configure a browser to route traffic through Burp Suite or ZAP, covering pre-configured browsers, manual proxy settings, FoxyProxy, and CA certificate installation.

## Tags
theory, proxy-setup, burp-suite, zap, foxyproxy, ca-certificate

## TL;DR — What's Important
- **Pre‑configured browsers are the fastest way to start:** Burp's *Open Browser* and ZAP's Firefox icon launch a browser already proxied with certificates installed.
- **Manual setup points the browser to 127.0.0.1:8080:** Both tools default to port 8080; changeable under Proxy Settings (Burp) or Network → Local Servers (ZAP).
- **FoxyProxy makes switching painless:** Install the Firefox extension, define the proxy (127.0.0.1:8080), and toggle it on/off from the toolbar — no digging through preferences.
- **The CA certificate is mandatory for clean HTTPS interception:** Without it, the browser will show certificate warnings for every HTTPS site or block traffic altogether.
- **Install the certificate once per tool:** Browse to `http://burp` for Burp's cert; in ZAP, go to *Tools → Options → Network → Server Certificates → Save*, then import into Firefox's Authorities and mark it trusted.

## Concept Overview
Before a web proxy can capture traffic, the browser must be told to send everything through it. You can either use the proxy's built‑in pre‑configured browser (zero‑setup) or manually configure a real browser like Firefox. Manual setup involves two steps: setting the proxy address/port (127.0.0.1:8080) and importing the proxy's CA certificate so HTTPS traffic can be decrypted seamlessly. The FoxyProxy extension simplifies switching, and the certificate installation is a one‑time import.

## Key Concepts

### Pre‑Configured Browsers
- **Burp Suite:** *Proxy → Intercept → Open Browser* launches a Chromium instance already proxied through Burp with certificates installed.
- **ZAP:** Click the Firefox icon in the top toolbar to launch a browser pre‑configured to proxy through ZAP.
- Best for quick tests; no configuration needed.

### Manual Proxy Setup (Firefox)
1. Go to Firefox network preferences or use **FoxyProxy**.
2. Set the proxy to `127.0.0.1` on port `8080` (default for both tools).
3. In FoxyProxy, give it a name (e.g., "Burp" or "ZAP"), save, and toggle it on.
4. This is already pre‑configured in PwnBox.

### CA Certificate Installation
- **Why:** Without it, HTTPS sites will show certificate errors because the proxy acts as a MITM presenting its own certificate.
- **Burp certificate:** With proxy enabled, browse to `http://burp` and click *CA Certificate* to download.
- **ZAP certificate:** *Tools → Options → Network → Server Certificates → Save*.
- **In Firefox:** Go to `about:preferences#privacy` → *View Certificates* → *Authorities* → *Import* → select the downloaded cert → tick *Trust this CA to identify websites* → OK.

## Why It Matters
A proxy that isn't receiving traffic is useless. HTTPS interception must be smooth — if every request triggers a certificate warning, testing grinds to a halt. FoxyProxy prevents the common mistake of leaving the proxy on and wondering why the internet is "broken" (because Burp/ZAP is closed). The certificate step is the number‑one setup gotcha for newcomers.

## Key Takeaways
- The pre‑configured browser is ideal for focused testing; a real browser with FoxyProxy is better for longer, real‑world simulations.
- Port 8080 is the default, but you can change it if it conflicts with another service.
- The CA certificate only needs to be imported once per browser profile; updates to the proxy tool may regenerate it.
- "Trust this CA to identify websites" must be checked, or HTTPS interception will still produce warnings.
- Remember to toggle FoxyProxy off when you're done proxying or the browser will be unusable without the proxy running.

## Gotchas
- If you change the proxy's listening port, you must update FoxyProxy (or the browser's manual settings) to match.
- Some browsers (e.g., Chrome/Chromium on Linux) use the system's certificate store by default, not their own — certificate installation might need to happen at the OS level instead.
- Leaving the proxy on with Burp/ZAP closed makes the browser unable to reach any website; FoxyProxy helps but you still need to remember to switch it off.
- The `http://burp` certificate download only works while Burp is running and the proxy is enabled in the browser.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-proxies]]  
← [[02-installation]] | [[04-intercepting-requests]] →
<!-- AUTO-LINKS-END -->
