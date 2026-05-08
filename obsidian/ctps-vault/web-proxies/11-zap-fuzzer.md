## ID
534

## Module
Using Web Proxies

## Kind
notes

## Title
Section 11 — ZAP Fuzzer

## Description
Using ZAP Fuzzer to fuzz web directories and cookie values with built-in wordlists, payload processors, and concurrent threading — no speed throttle unlike free Burp Intruder.

## Tags
zap, fuzzer, fuzzing, cookies, md5, wordlists

## Commands
- zaproxy (or owasp-zap) to launch ZAP from terminal
- Right-click request → Attack > Fuzz
- Payload type: File Fuzzers (built-in wordlists)
- Payload type: File (custom wordlist path)
- Processor: URL Encode
- Processor: MD5 Hash
- /opt/useful/seclists/Usernames/top-usernames-shortlist.txt
- curl -x http://127.0.0.1:<PORT> http://<TARGET_IP>:<PORT> to test proxy
- ss -tlnp | grep <PORT> to verify ZAP is listening

## What This Section Covers
ZAP Fuzzer is ZAP's equivalent to Burp Intruder but without speed throttling, making it practical for real fuzzing tasks on the free version. It supports payload processors like hashing and encoding, letting you transform wordlist entries before sending — critical for attacks like fuzzing MD5-hashed cookie values.

## Methodology
1. Capture a request in ZAP's Proxy History
2. Right-click → `Attack > Fuzz` to open the Fuzzer window
3. Select the word you want to fuzz in the request, click `Add` to set the **Fuzz Location**
4. In the Payloads window, click `Add` → choose payload type (`File` for custom wordlist, `File Fuzzers` for built-in)
5. Add **Processors** as needed (URL Encode, MD5 Hash, Prefix/Postfix String, etc.)
6. In **Options**, set `Concurrent Scanning Threads` (e.g. 20) for speed
7. Click `Start Fuzzer`, sort results by Response Code or Size to find hits

## ZAP Setup & Proxy Configuration
1. **Launch ZAP**: run `zaproxy` or `owasp-zap` from terminal, or go to Applications → Web Application Analysis → OWASP ZAP
2. **Check ZAP's proxy port**: Tools → Options → Network → Local Servers/Proxies (default is `127.0.0.1:8080`)
3. **If Burp is already on 8080**, ZAP may auto-switch to `8081` — pick one port and make everything match
4. **Browser proxy setup (Firefox, no FoxyProxy needed)**: Firefox → Settings → search "proxy" → Settings → Manual proxy configuration → HTTP Proxy `127.0.0.1`, Port `8080` (or whatever ZAP is on) → check "Also use this proxy for HTTPS" → clear the "No proxy for" field → OK
5. **Or use ZAP's built-in browser**: click Manual Explore → enter target URL → Launch Browser (already pre-configured, no proxy settings needed)
6. **If using proxychains**, edit `/etc/proxychains.conf` and make sure the port matches ZAP: `http 127.0.0.1 8080`
7. **Verify ZAP is listening**: `ss -tlnp | grep 8080`
8. **Test manually**: `curl -x http://127.0.0.1:8080 http://TARGET_IP:PORT/` — check if it appears in ZAP History

## Multi-step Workflow — Lab Walkthrough (Fuzz MD5 cookie for usernames)
1. Open ZAP's built-in browser (Manual Explore → Launch Browser) and visit `http://TARGET_IP:PORT/skills/`
2. Check ZAP History — the **first** request won't have a cookie, but the **response** will have a `Set-Cookie` header with the MD5 hash of "guest"
3. **Refresh the page** (`F5`) — this second request now **sends** the cookie back in the request header, looking like `Cookie: cookie=e8b32bc4d7b564ac6075a1418ad8841e`
4. Right-click this second request (the one with the cookie) → `Attack > Fuzz`
5. In the request pane, select the MD5 hash value in the `Cookie` header and click `Add` to set it as the Fuzz Location
6. In the Payloads window, click `Add` → set Type to `File`
7. Browse to `/opt/useful/seclists/Usernames/top-usernames-shortlist.txt` and load it
8. Click `Add` under **Processors** → select `MD5 Hash` so each username gets hashed before being placed in the cookie
9. Click `Generate Preview` to confirm payloads look like MD5 hashes
10. Click `Add` to save the processor, then `OK` to close the Payloads window
11. In **Options**, set `Concurrent Scanning Threads` to 20
12. Click `Start Fuzzer`
13. Sort results by `Size Resp. Body` — the response with a different size than the others contains the flag
14. Click on that result to view the response body and grab the flag

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Fuzz the cookie with MD5-hashed usernames to get the flag | Check the response | Fuzz the cookie value using `top-usernames-shortlist.txt` with MD5 Hash processor, sort by response size, the unique one has the flag |

## Key Takeaways
- ZAP Fuzzer has no speed throttle — far more practical than free Burp Intruder for real-world fuzzing
- **Payload Processors** transform each wordlist entry before sending — MD5 Hash, URL Encode, SHA-256, Prefix/Postfix String, or custom scripts
- Built-in wordlists (`File Fuzzers` type) mean you can fuzz without providing your own lists; more can be installed from ZAP Marketplace
- **Depth First** = try all payloads on one position before moving to the next; **Breadth First** = try one payload across all positions before moving to the next word
- Sort by `Response Code`, `Size Resp. Body`, or `RTT` depending on what indicates a hit (200 vs 404, different page size, time delay for SQLi)
- The concurrent threads setting directly controls speed but is limited by your hardware and the server's connection tolerance

## Gotchas
- **Port mismatch**: If ZAP is on 8080 but proxychains or Firefox points to 8081 (or vice versa), nothing gets captured — always verify they match
- **No cookie on first request**: The first visit to `/skills/` only gets a `Set-Cookie` in the response — you must **refresh** (`F5`) so the second request actually sends the cookie in the request header
- **Burp stealing port 8080**: If Burp is already running on 8080, ZAP silently switches to 8081 — check Tools → Options → Network → Local Servers/Proxies to confirm
- **Empty History tab**: Means your browser isn't proxying through ZAP — either configure Firefox manually or use ZAP's built-in browser via Manual Explore → Launch Browser
- **FoxyProxy not required**: You can set the proxy directly in Firefox settings (Settings → search "proxy" → Manual proxy configuration) without any extension


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-proxies]]  
← [[10-burp-intruder]] | [[12-burp-scanner]] →
<!-- AUTO-LINKS-END -->
