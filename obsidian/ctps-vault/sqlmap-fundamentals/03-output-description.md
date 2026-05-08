## ID
302

## Module
SQLMap Essentials

## Kind
notes

## Title
Section 3 — SQLMap Output Description

## Description
Explains the most common log messages produced by SQLMap during a scan, helping you interpret results and understand the tool’s automated decision‑making.

## Tags
sqlmap, output, interpretation, theory, scan, log

## TL;DR — What's Important
- **“target URL content is stable”** – Responses to identical requests are consistent, making injection detection reliable.
- **“parameter appears to be dynamic”** – The parameter causes changes in the response, a prerequisite for most SQLi.
- **“heuristic (basic) test shows that GET parameter … might be injectable”** – A quick check using an invalid value indicates a possible injection and hints at the DBMS.
- **“parameter appears to be … injectable”** – SQLMap has found a working injection type; the message includes the technique and optional `--string` for true/false distinction.
- **“sqlmap identified the following injection point(s) with a total of … HTTP(s) requests”** – Final proof: the listed payloads are exploitable.
- **Session data is saved locally** – All logs, sessions, and output are stored under `~/.sqlmap/output/<domain>/`, allowing faster subsequent runs.

## Concept Overview
SQLMap’s output stream is deliberately verbose, reporting every stage of detection and exploitation. Understanding these messages is essential for interpreting what the tool found, whether it’s a false positive, and how to manually verify or extend the attack. Each message signals a specific check or decision, from initial stability tests to the final list of exploitable injection points.

## Key Concepts

### Common Log Messages and Their Meaning
- **“target URL content is stable”**  
  Responses are consistent across identical requests. Stability allows SQLMap to detect subtle changes caused by injected payloads without noise.

- **“GET parameter 'id' appears to be dynamic”**  
  The parameter’s value influences the response. If the output were static, the parameter likely isn’t processed by the backend in that context.

- **“heuristic (basic) test shows that GET parameter 'id' might be injectable (possible DBMS: 'MySQL')”**  
  Sending an intentionally malformed value (like `?id=1",)..)'))`) triggered a recognizable DBMS error. This is only an indicator, not proof; further tests will confirm.

- **“heuristic (XSS) test shows that GET parameter 'id' might be vulnerable to cross‑site scripting (XSS) attacks”**  
  SQLMap runs a quick XSS check. Useful in large scans even if the primary goal is SQLi.

- **“it looks like the back‑end DBMS is 'MySQL'. Do you want to skip test payloads specific for other DBMSes? [Y/n]”**  
  After fingerprinting the DBMS, SQLMap asks whether to skip tests for other database systems, speeding up the scan.

- **“for the remaining tests, do you want to include all tests for 'MySQL' extending provided level (1) and risk (1) values? [Y/n]”**  
  If the DBMS is identified, SQLMap offers to expand the test set for that DBMS beyond the default level/risk, increasing detection chances.

- **“reflective value(s) found and filtering out”**  
  Parts of the payload appear in the response. SQLMap automatically filters this junk to prevent false comparisons.

- **“GET parameter 'id' appears to be 'AND boolean‑based blind - WHERE or HAVING clause' injectable (with --string="luther")”**  
  A specific injection type was found. The `--string` value indicates a marker in the TRUE response that SQLMap uses to distinguish true/false, greatly reducing false positives.

- **“time‑based comparison requires larger statistical model, please wait........... (done)”**  
  SQLMap collects enough baseline response times to build a statistical model, allowing it to reliably detect intentional delays even in high‑latency environments.

- **“automatically extending ranges for UNION query injection technique tests as there is at least one other (potential) technique found”**  
  Because another injection type was detected, SQLMap invests more requests into UNION checks, improving the chance of finding a usable UNION payload.

- **“ORDER BY' technique appears to be usable. This should reduce the time needed to find the right number of query columns. Automatically extending the range for current UNION query injection technique test”**  
  The `ORDER BY` heuristic works, so SQLMap will use a binary‑search approach to quickly determine the number of columns required for the UNION.

- **“GET parameter 'id' is vulnerable. Do you want to keep testing the others (if any)? [y/N]”**  
  At least one exploitable injection point has been confirmed. You can stop here or continue to find more parameters.

- **“sqlmap identified the following injection point(s) with a total of 46 HTTP(s) requests:”**  
  Final summary listing all proven injectable parameters, with type, title, and payload. Only exploitable points are shown.

- **“fetched data logged to text files under '/home/user/.sqlmap/output/www.example.com'”**  
  All session and output data are stored locally. Subsequent runs reuse this information to minimize requests.

## Why It Matters
Correct interpretation of SQLMap’s output is crucial for verifying findings, tuning future attacks, and reporting with confidence. Knowing what each message means lets you distinguish between a promising heuristic and a confirmed vulnerability, avoid false positives, and understand the tool’s limitations.

## Defender Perspective
- **Detection:** Security tools can flag the presence of SQLMap through its verbose user‑agent or by observing the sequence of malformed requests that produce DBMS errors. The tool’s behavior—testing for stability, then dynamic injection, then specific payloads—leaves a recognizable traffic pattern.
- **Mitigations:** The same as for general SQLi: parameterised queries, input validation, and minimal error disclosure. Disabling detailed error messages removes the quick heuristic indicator SQLMap relies on.
- **MITRE ATT&CK:** T1190 (Exploit Public‑Facing Application), T1082 (System Information Discovery) when DBMS info is leaked.

## Key Takeaways
- Not every “might be injectable” message means a real vulnerability; final confirmation comes with the “is vulnerable” line and listed payloads.
- The `--string` indicator is a strong signal that the detection is reliable because SQLMap found a consistent true/false marker.
- SQLMap automatically extends tests for UNION and other techniques if it finds at least one potential injection, improving the chances of discovering the fastest exploitation path.
- Session data stored locally is valuable: you can resume partial scans or re‑exploit without repeating initial detection steps.
- If you see “reflective value(s) found and filtering out”, trust that SQLMap is handling response noise, but you may still want to manually verify with `--no-cast` in edge cases.

## Gotchas
- A “static” parameter message does not necessarily mean no injection is possible; the parameter might be used in a different context (e.g., POST, header) or require specific values to trigger processing.
- The heuristic DBMS guess can be wrong; always let SQLMap complete the full test or manually specify with `--dbms` if you’re certain.
- “Time‑based comparison requires larger statistical model” can take a long time on high‑latency networks; be patient—it’s building the baseline needed for accurate detection.
- The output directory path `~/.sqlmap/output/` contains session files that include sensitive data (target URLs, payloads, extracted data); clean it if sharing the environment.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sqlmap-fundamentals]]  
← [[02-getting-started]] | [[04-http-request]] →
<!-- AUTO-LINKS-END -->
