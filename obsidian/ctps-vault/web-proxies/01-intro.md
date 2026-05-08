## ID
200

## Module
Using Web Proxies

## Kind
notes

## Title
Section 1 — Intro to Web Proxies

## Description
Introduces web proxies as essential man-in-the-middle tools for capturing, viewing, and manipulating HTTP/HTTPS traffic, and compares the two leading tools: Burp Suite and OWASP ZAP.

## Tags
theory, web-proxy, burp-suite, zap, introduction, concept

## TL;DR — What's Important
- **Web proxies sit between client and server:** They capture, inspect, and modify HTTP/HTTPS requests and responses, acting as a controlled MITM.
- **They are the Swiss Army knife of web app pentesting:** Used for analysis, fuzzing, crawling, scanning, mapping, and replaying requests.
- **The two main tools are Burp Suite and OWASP ZAP:** Burp has a polished commercial edition; ZAP is entirely open‑source with no feature paywalls.
- **No single tool is best for every situation:** Burp Pro offers advanced automation, while ZAP provides unlimited scanning without throttling; learning both gives flexibility.
- **Web proxies focus on web ports (80, 443):** Unlike general sniffers like Wireshark, they are specialised for HTTP‑layer analysis and modification.

## Concept Overview
Web proxies are specialised tools that intercept web traffic between a browser/mobile app and the back‑end server, enabling testers to examine and tamper with the entire communication flow. They are fundamental to modern web application penetration testing, simplifying tasks that once required command‑line tools. By configuring the browser to route through the proxy, every request can be captured, held (intercepted), modified, or replayed — turning passive observation into active manipulation.

## Key Concepts

### What a Web Proxy Does
- **Captures requests and responses:** View headers, cookies, parameters, bodies — all in plain text for HTTP and decrypted for HTTPS (via user‑trusted CA).
- **Intercepts traffic:** Pause a request before it reaches the server, edit any part of it (URL, parameter, header), then forward the modified version.
- **Replays requests:** Send a captured request repeatedly (or with variations) without using the browser — essential for fuzzing and brute‑forcing.

### Common Use Cases
- Vulnerability scanning (automated or manual)
- Web fuzzing (injecting payloads into parameters)
- Web crawling and spidering to map all endpoints
- Request analysis and debugging
- Security configuration testing (e.g., testing HTTPS, CORS, CSP)
- Assisting code reviews by observing the real traffic patterns

### Toolkit Comparison
| Feature | Burp Suite (Community) | Burp Suite (Pro) | OWASP ZAP |
|---------|------------------------|------------------|-----------|
| **Cost** | Free | Paid (trial available) | Free + open‑source |
| **Interceptor / Repeater** | Yes | Yes | Yes |
| **Automated Scanner** | Limited (manual) | Active web app scanner | Active scanner (no throttling) |
| **Intruder (Fuzzing / Brute‑force)** | Slow (throttled) | Fast (multi‑threaded) | Fuzzer (unlimited speed) |
| **Extensions** | BApp store (some pro‑only) | All extensions | Marketplace (all free) |
| **Built‑in Browser** | Yes (Chromium) | Yes | No (but integrates with any browser) |
| **Collaborator** (out‑of‑band) | No | Yes | No (but can use external OAST) |

## Why It Matters
Every web application pentest begins with understanding the traffic flow. Without a proxy, you are blind to what the application actually sends and receives — you cannot test what you cannot see. The ability to intercept and modify requests is the foundation for virtually all manual web attacks (SQLi, XSS, CSRF, IDOR, etc.). Knowing both Burp and ZAP ensures you are never limited by tool availability or licensing restrictions on an engagement.

## Defender Perspective
- **Detection:** Security teams monitor network traffic for signs of proxy interception (e.g., unusual certificates, extra headers added by proxies, or traffic to proxy default ports). Some WAFs can fingerprint requests modified by known proxy tools.
- **Mitigations:** Certificate pinning in mobile apps can prevent easy decryption; however, in a testing context, the tester usually controls the device. For server‑side, defence is less about blocking proxies and more about hardening the application against the attacks a proxy enables.
- **MITRE ATT&CK:** While not directly mapped, web proxies facilitate techniques like *T1071 – Application Layer Protocol* (for C2), *T1566 – Phishing* (crafting malicious requests), and *T1592 – Gather Victim Host Information* (mapping the app).

## Key Takeaways
- A web proxy is the tester’s primary lens — without it, you cannot reliably perform manual injection testing.
- Intercept abilities distinguish proxies from mere packet sniffers: you can actively change traffic, not just watch it.
- Burp’s free version is powerful but intentionally throttled on fuzzing; ZAP has no such limits. Switching between the two can be a tactical choice.
- The proxy must install its own CA certificate in the browser/OS to decrypt HTTPS; forgetting this step is the #1 setup mistake.
- Even automated features (scanners, fuzzers) should be understood and verified manually — proxies give you the visibility to do that.

## Gotchas
- If the target application uses certificate pinning (common in mobile apps), the proxy will not be able to decrypt HTTPS traffic unless the pinning is bypassed first.
- Using a proxy with an intercepting feature left on can make the application appear “broken” (requests hang). Always check your intercept toggle first.
- Burp’s community edition intruder is intentionally rate‑limited, so time‑sensitive brute‑force attacks may need ZAP or a custom script instead.
- Some sites detect and block known proxy user‑agent strings or headers (e.g., `BurpSuite`). Always match the browser’s normal fingerprint where needed.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-proxies]]  
[[02-installation]] →
<!-- AUTO-LINKS-END -->
