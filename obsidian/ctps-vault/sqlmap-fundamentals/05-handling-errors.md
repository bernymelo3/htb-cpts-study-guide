## ID
303

## Module
SQLMap Essentials

## Kind
notes

## Title
Section 5 — Handling SQLMap Errors

## Description
Explains practical methods for debugging SQLMap issues: displaying DBMS errors, saving traffic to a file, increasing verbosity, and routing through a proxy.

## Tags
sqlmap, errors, debugging, verbose, proxy, traffic

## Commands
- `sqlmap -u "<URL>" --parse-errors` — display parsed DBMS errors in real time
- `sqlmap -u "<URL>" --batch -t /tmp/traffic.txt` — store all HTTP traffic to a file for inspection
- `sqlmap -u "<URL>" -v 6 --batch` — set verbosity to highest level (shows full requests and responses)
- `sqlmap -u "<URL>" --proxy="http://127.0.0.1:8080"` — route all traffic through a web proxy like Burp/ZAP
- `cat /tmp/traffic.txt` — view captured requests/responses

## What This Section Covers
When SQLMap encounters unexpected behavior or fails to detect injection, you need debugging tools to see exactly what’s happening. This section teaches four key flags: `--parse-errors` to surface database error messages, `-t` to write full HTTP traffic to a file, `-v` for detailed console output, and `--proxy` to pass everything through your interception proxy for manual inspection.

## Methodology
1. Use `--parse-errors` to see any DBMS error messages that the target returns. These often reveal syntax issues and help you adjust payloads.
2. Save traffic with `-t <file>` to record all requests and responses. Then open the file to manually review the exact exchange.
   ```bash
   sqlmap -u "http://www.target.com/vuln.php?id=1" --batch -t /tmp/traffic.txt
   cat /tmp/traffic.txt

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sqlmap-fundamentals]]  
← [[04-http-request]] | [[06-attack-tuning]] →
<!-- AUTO-LINKS-END -->
