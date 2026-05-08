## ID
206

## Module
Using Web Proxies

## Kind
notes

## Title
Section 12 — Burp Scanner

## Description
Explores Burp Suite’s Pro-only web vulnerability scanner: defining scope, running crawl/passive/active scans, interpreting issues, and generating reports.

## Tags
burp-suite, scanner, pro-feature, crawl, active-scan, passive-scan

## Commands
- Right-click request in Proxy History → **Scan** to configure and launch a scan
- Right-click target in **Target → Site map** → **Add to scope**
- **Dashboard → New Scan** → choose *Crawl* or *Crawl and Audit*
- Select a preset from **Select from library** (e.g., *Crawl strategy - fastest*, *Audit checks - critical issues only*)
- Right-click item → **Do passive scan** (passive only)
- Right-click item → **Do active scan** (active only)
- In **Target → Site map**, right-click host → **Issue → Report issues for this host** to export findings

## What This Section Covers
Burp Suite Professional includes a built-in web vulnerability scanner that crawls, passively analyses, and actively tests web applications for a wide range of issues. This section walks through setting a target scope, running crawl-only and full crawl‑and‑audit scans, understanding passive vs. active scanning, filtering results by severity/confidence, and exporting a structured report.

## Methodology
1. **Set the target scope**  
   - Browse the target through Burp so requests appear in **Proxy History** or **Target > Site map**.  
   - Right-click the target host/folder and select **Add to scope**.  
   - Burp prompts to restrict all activity to in‑scope items; accept to avoid noise.
2. **Run a crawl (optional but recommended)**  
   - Go to **Dashboard** > **New Scan**.  
   - The URL field auto‑fills with in‑scope items; select **Crawl**.  
   - In **Scan configuration**, click **Select from library** and pick *Crawl strategy - fastest* (or a custom config).  
   - Click **Ok** to start. Monitor progress in **Dashboard > Tasks**.
3. **Review the site map**  
   - After crawling, **Target > Site map** shows all discovered pages and endpoints.
4. **Passive scan**  
   - Right-click the target in the site map or a request in history → **Do passive scan**.  
   - Passive scanning analyses existing responses without sending new requests.  
   - Results appear in the **Issue activity** pane; note *Severity* and *Confidence*.
5. **Active scan**  
   - For a full test, select **Crawl and Audit** when creating a new scan, or right-click an item → **Do active scan**.  
   - In **Audit configuration**, you can choose presets like *Audit checks - critical issues only* to focus on high‑impact vulnerabilities.  
   - The active scanner sends crafted payloads, fuzzes parameters, and runs JavaScript analysis.  
   - Monitor the **Logger** tab to see all generated requests.
6. **Interpret results**  
   - After the scan, filter issues by **Severity: High** and **Confidence: Certain / Firm**.  
   - Click an issue to view its advisory, proof‑of‑concept, and remediation advice.  
   - Example: a firm OS command injection in an `ip` parameter.
7. **Export a report**  
   - Right-click the target in **Target > Site map** → **Issue → Report issues for this host**.  
   - Choose export type (HTML/XML) and desired detail level. The report includes issue counts, descriptions, and remediation.

## Key Takeaways
- **Scope is crucial:** Define exactly what Burp should scan to avoid unintended side effects (e.g., logout links) and save resources.
- **Passive scans are safe and quick** but only flag potential issues; active scans verify them with actual attacks.
- **Active scans are powerful but intrusive** – they can modify data, trigger alerts, or break application state; always use with permission and on isolated test environments.
- **The crawler builds the site map**, which feeds the scanner; missing pages beyond the crawl scope won’t be analysed.
- **Confidence levels** help triage: “Certain”/”Firm” findings are reliable; “Tentative” may require manual verification.
- **Burp’s report is a supplement, not a final deliverable** – it should be integrated into a human‑written report with business‑context analysis.

## Gotchas
- Burp Scanner is **Pro/Enterprise only**; the Community edition cannot perform active or passive vulnerability scans.
- The crawler follows links but does **not** brute‑force directories like ffuf/dirbuster; undiscovered endpoints remain untested.
- Active scanning can accidentally trigger destructive actions (e.g., creating/deleting resources). Always exclude sensitive functions (logout, admin‑level CRUD) from scope or run in a dedicated test environment.
- Large scans can take hours; use the “critical issues only” preset for faster high‑value results when time is limited.
- Leaving a scan unmonitored may hit request‑rate limits on production servers or cause WAF blocks; adjust crawl strategy and throttle settings accordingly.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-proxies]]  
← [[11-zap-fuzzer]] | [[13-zap-scanner]] →
<!-- AUTO-LINKS-END -->
