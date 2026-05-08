## ID
207

## Module
Using Web Proxies

## Kind
notes

## Title
Section 14 — Extensions

## Description
Covers how to extend Burp Suite and ZAP functionality through their respective marketplaces (BApp Store and ZAP Marketplace), including installing and using selected extensions.

## Tags
extensions, burp-suite, zap, bapp-store, marketplace, plugins

## Commands
- In Burp, go to **Extensions → BApp Store** to browse and install extensions.
- Select an extension (e.g., `Decoder Improved`) and click **Install**.
- In ZAP, click **Manage Add-ons** → **Marketplace** to view available add-ons.
- Install `FuzzDB Files` and `FuzzDB Offensive` to gain additional fuzzing wordlists.
- In ZAP fuzzer, after installation, select payload type **File Fuzzers** and pick wordlist from `fuzzdb > attack > os-cmd-execution`.

## What This Section Covers
Both Burp Suite and OWASP ZAP support a plugin architecture that lets the community develop and share extensions. This section shows how to access Burp’s BApp Store and ZAP’s Marketplace and install popular extensions that add encoding capabilities or specialised wordlists, enhancing your testing efficiency.

## Methodology
1. Open Burp Suite and go to the **Extensions** tab.
2. Switch to the **BApp Store** sub‑tab to view all available extensions.
3. Sort by **Popularity** to find highly‑rated extensions.
4. Click on an extension (e.g., `Decoder Improved`) to read about it, then click **Install**.
5. Some extensions require additional components like Jython; install them if prompted.
6. Once installed, the extension adds a new tab (e.g., `Decoder Improved`) that you can use immediately — for instance, to hash text with MD5 via its interface.
7. In ZAP, click the **Manage Add‑ons** button on the toolbar.
8. Switch to the **Marketplace** tab to list add‑ons; note the release status (Release, Beta, Alpha).
9. Select `FuzzDB Files` and `FuzzDB Offensive` (both Release) and install them.
10. Return to the ZAP fuzzer; now, under **File Fuzzers** payload type, you’ll find additional wordlists, such as `fuzzdb/attack/os-cmd-execution/command_execution-unix.txt`.
11. Run a fuzzing attack against the ping exercise using this wordlist to see multiple injection variants that bypass basic filters.

## Key Takeaways
- Extensions dramatically expand what your proxy can do — from additional encoding helpers to specialised vulnerability scanners.
- Burp’s BApp Store has many Pro‑only extensions; check compatibility before installing.
- ZAP’s Marketplace add‑ons are all free and open‑source, with clear release status indicators.
- FuzzDB integration in ZAP gives you immediate access to extensive payload lists without leaving the tool.
- Always review an extension’s documentation (linked in the store) to understand its full capabilities.
- Closing thoughts: Burp Suite and ZAP are essential tools for any web penetration tester; this module has equipped you with the fundamentals, but further practice on HTB boxes and web‑specific Academy modules will solidify these skills.

## Gotchas
- Extensions may conflict with each other or with certain Burp/ZAP versions; if you experience strange behaviour, disable recent extensions to isolate the cause.
- Some Burp extensions require Jython (Python for Java) or JRuby — missing dependencies will prevent installation.
- ZAP add‑ons marked as **Beta** or **Alpha** may cause instability or crash the application; use them only in testing environments.
- Installing large wordlist packs (like FuzzDB) can increase memory usage and startup time.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-proxies]]  
← [[13-zap-scanner]] | [[skills-assessment]] →
<!-- AUTO-LINKS-END -->
