## ID
535

## Module
Using Web Proxies

## Kind
notes

## Title
Section 13 — ZAP Scanner

## Description
Using ZAP's Spider, Passive Scanner, and Active Scanner to automatically discover directories, map site structure, and identify vulnerabilities like OS command injection.

## Tags
zap, scanner, spider, active-scan, passive-scan, command-injection

## Commands
- Right-click request → Attack > Spider
- Right-click request → Attack > Active Scan
- Report > Generate HTML Report
- cat /flag.txt (via command injection)

## What This Section Covers
ZAP Scanner combines spidering (crawling for links), passive scanning (analyzing responses for issues like missing headers), and active scanning (sending attack payloads to find vulnerabilities). Together they automate the discovery phase and vulnerability identification that would take hours manually.

## Methodology
1. Browse to the target to capture a request in History
2. Run **Spider** to crawl the site and build a site tree of all pages and directories
3. **Passive Scanner** runs automatically during the spider — flags issues from response source code (missing headers, DOM XSS, etc.)
4. Run **Active Scanner** on the built site tree — it sends attack payloads against all discovered pages and parameters
5. Review **Alerts** sorted by severity (High → Low → Informational)
6. Investigate High alerts — click to see the exact request/response and how to reproduce
7. Generate a report via `Report > Generate HTML Report`

## Multi-step Workflow — Lab Walkthrough (Find and exploit the High vulnerability)
1. Launch ZAP and open the built-in browser (Manual Explore → enter `http://TARGET_IP:PORT` → Launch Browser)
2. Visit the target page so it appears in ZAP History
3. Right-click the request in History → `Attack > Spider` (if ZAP asks to add the site to Scope, click Yes)
4. Wait for the Spider to finish — check the **Sites** tab on the left to see all discovered pages and directories
5. Optionally run **Ajax Spider** (third button on right pane in HUD) for JavaScript-loaded links the normal spider might miss
6. Click the **Active Scan** button (right pane in HUD) or right-click the site in Sites tab → `Attack > Active Scan`
7. Wait for the Active Scan to complete — watch the Alerts count increase as ZAP finds issues
8. Click on the **High Alerts** — you should see **Remote OS Command Injection**
9. Click the alert to see details — note the vulnerable parameter and the attack payload ZAP used (e.g. `127.0.0.1&cat /etc/passwd&`)
10. Now exploit it manually: find the vulnerable page (likely a ping/IP input form)
11. Enter a command injection payload to read the flag: `127.0.0.1&cat /flag.txt&` or `127.0.0.1;cat /flag.txt;`
12. You can do this through the browser, or use ZAP's Request Editor to replay the request with the payload in the parameter
13. The flag will be in the response

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Find the high-level vulnerability and read /flag.txt | Check the response | Run Spider → Active Scan → find Remote OS Command Injection alert → inject `127.0.0.1;cat /flag.txt` in the vulnerable parameter |

## Key Takeaways
- **Spider** crawls links to build a site tree; **Ajax Spider** additionally catches JavaScript AJAX requests — run both for full coverage
- **Passive Scanner** runs automatically on every response during spidering — no extra setup needed, flags things like missing `X-Frame-Options`, CSP headers, etc.
- **Active Scanner** sends actual attack payloads (SQLi, XSS, command injection, etc.) — takes longer but finds exploitable vulnerabilities
- Always check **High alerts first** — these usually lead to direct compromise (RCE, SQLi, etc.)
- Click any alert to see the exact request, response, and payload ZAP used — you can replay it directly from there
- **Scope** controls which sites get scanned — ZAP will prompt to add a site to scope before scanning; you can manually add multiple targets
- Reports can be exported as HTML, XML, or Markdown — useful for documentation during pentests

## Gotchas
- ZAP may auto-add the site to scope when you start a Spider — click Yes or the scan won't start
- Active Scan takes significantly longer than Spider — don't kill it early or you'll miss vulnerabilities
- ZAP's HUD may not work properly in some browser versions — if buttons don't respond, use the main ZAP UI instead (right-click in History/Sites tab)
- The command injection payload format matters — try both `&cat /flag.txt&` and `;cat /flag.txt;` depending on how the server handles input


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-proxies]]  
← [[12-burp-scanner]] | [[14-extensions]] →
<!-- AUTO-LINKS-END -->
