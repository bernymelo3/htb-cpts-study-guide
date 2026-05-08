## ID
533

## Module
Using Web Proxies

## Kind
notes

## Title
Section 10 — Burp Intruder

## Description
Using Burp Intruder to fuzz web directories, files, and parameters by configuring payload positions, wordlists, processing rules, and grep-match filters.

## Tags
burp, intruder, fuzzing, brute-force, wordlists, web-proxy

## Commands
- Ctrl+I to send request to Intruder
- Ctrl+Shift+I to switch to Intruder tab
- /opt/useful/seclists/Discovery/Web-Content/common.txt
- Skip if matches regex: ^\..*$
- Grep - Match: 200 OK

## What This Section Covers
Burp Intruder is a built-in web fuzzer that replaces CLI tools like ffuf or gobuster for directory/file/parameter fuzzing. You mark a payload position in a captured request, load a wordlist, and Intruder iterates through each entry. The free version is throttled to ~1 req/sec, so it's only practical for short wordlists.

## Methodology
1. Capture a request in Proxy History, right-click → `Send to Intruder` or `Ctrl+I`
2. In **Positions**, highlight the fuzz target (e.g. the directory path) and click `Add §` to mark it as the payload position
3. In **Payloads**, select `Simple List` as Payload Type, click `Load` and choose your wordlist (e.g. `common.txt`)
4. Optionally add **Payload Processing** rules (e.g. `Skip if matches regex` with `^\..*$` to skip dotfiles)
5. In **Settings → Grep - Match**, clear defaults, add `200 OK`, and disable `Exclude HTTP Headers` so status codes get flagged
6. Click `Start Attack`, then sort results by the `200 OK` column or by `Status` to find hits

## Multi-step Workflow — Lab Walkthrough (Fuzz .html under /admin)
1. Open Burp's pre-configured browser and visit `http://TARGET_IP:PORT/admin/`
2. Go to `Proxy > HTTP History`, find the GET request to `/admin/`
3. Right-click it → `Send to Intruder` (or `Ctrl+I`)
4. In the `Intruder` tab (`Ctrl+Shift+I`), click `Clear §` to remove any auto-highlighted positions
5. Edit the request path so it reads `GET /admin/§value§.html HTTP/1.1` — select the word `value` and click `Add §` so Intruder injects wordlist entries there, with `.html` hardcoded after
6. On the right side under **Payloads**, set Payload Type to `Simple List`
7. Click `Load` and select `/opt/useful/seclists/Discovery/Web-Content/common.txt`
8. Under **Payload Processing**, click `Add` → select `Skip if matches regex` → enter `^\..*$` → OK (skips dotfile entries)
9. Go to the **Settings** tab, scroll to **Grep - Match**, enable it
10. Click `Clear` to remove defaults, type `200 OK`, click `Add`, and uncheck `Exclude HTTP Headers`
11. Click `Start Attack` and wait for the scan to finish (slow on free version)
12. Sort results by the `200 OK` column — the entry with a checkmark is your hit
13. Note the filename from the `Payload` column of the matching result
14. Visit `http://TARGET_IP:PORT/admin/<filename>.html` in your browser — the flag is on the page

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Fuzz for .html files under /admin to find the flag | flag.html contains the flag | Set Intruder position to `GET /admin/§value§.html`, load common.txt, run attack, sort by 200 OK, visit the matching page |

## Key Takeaways
- Free Burp Intruder is capped at ~1 req/sec — only useful for short/targeted fuzzes; use CLI tools for large wordlists
- **Sniper** attack type = one payload position, one wordlist; **Cluster Bomb** = multiple positions with different wordlists
- Use `Runtime file` instead of `Simple List` for very large wordlists to avoid memory issues
- Payload Processing lets you skip, prefix, suffix, or regex-filter wordlist entries before they're sent
- Grep - Match flags responses containing your string (e.g. `200 OK`) so you can sort results instantly
- Grep - Extract pulls specific parts from responses — useful when you only care about a snippet in long responses
- Intruder works for directory fuzzing, file fuzzing, parameter brute-forcing, password spraying (OWA, VPN portals, RDS), and more


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-proxies]]  
← [[09-proxying-tools]] | [[11-zap-fuzzer]] →
<!-- AUTO-LINKS-END -->
