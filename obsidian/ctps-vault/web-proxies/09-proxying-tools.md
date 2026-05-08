## ID
532

## Module
Using Web Proxies

## Kind
notes

## Title
Section 9 — Proxying Tools

## Description
Routing command-line tools and Metasploit modules through Burp/ZAP using proxychains and the PROXIES option to inspect and debug their HTTP traffic.

## Tags
burp, zap, proxychains, metasploit, proxy, cli-tools

## Commands
- sudo nano /etc/proxychains.conf
- proxychains -q curl http://<TARGET_IP>:<PORT>
- msfconsole
- set PROXIES HTTP:127.0.0.1:8080
- use auxiliary/scanner/http/robots_txt
- use auxiliary/scanner/http/http_put
- set RHOST <TARGET_IP>
- set RPORT <PORT>

## What This Section Covers
Beyond browser traffic, you can route any CLI tool or framework through your web proxy to see exactly what it sends and receives. This is critical for debugging exploits, understanding scanner behavior, and modifying tool-generated requests on the fly.

## Methodology
1. **Proxychains setup**: Edit `/etc/proxychains.conf`, comment out `socks4 127.0.0.1 9050`, add `http 127.0.0.1 8080`
2. Prepend `proxychains -q` to any CLI command (e.g. `proxychains -q curl http://<TARGET>`) — the `-q` flag suppresses connection noise
3. **Metasploit setup**: Inside `msfconsole`, run `set PROXIES HTTP:127.0.0.1:8080` after selecting your module
4. Set `RHOST` and `RPORT`, then `run` — all HTTP traffic flows through Burp/ZAP
5. Check Proxy History in Burp/ZAP to inspect every request the tool made

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Run `auxiliary/scanner/http/http_put` through Burp. What is the last line in the request? | msf test file | Load the module, `set PROXIES HTTP:127.0.0.1:8080`, set RHOST to any target, `run`, then check the PUT request body in Burp's Proxy History |

## Key Takeaways
- Proxychains is the universal method — works with any CLI tool without the tool needing its own proxy setting
- Metasploit has a built-in `PROXIES` option (format: `HTTP:127.0.0.1:8080`) — no proxychains needed
- Only proxy tools when you need to inspect traffic; proxying slows them down
- This technique lets you replay and modify tool-generated requests using Repeater, combining the automation of scanners with manual precision
- The same approach works for thick clients and custom scripts — just point their proxy setting to `127.0.0.1:8080`


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-proxies]]  
← [[08-encoding-decoding]] | [[10-burp-intruder]] →
<!-- AUTO-LINKS-END -->
