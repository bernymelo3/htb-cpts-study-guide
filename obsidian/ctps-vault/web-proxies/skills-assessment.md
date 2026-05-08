# Skills Assessment — Using Web Proxies

## ID
531

## Module
Using Web Proxies

## Kind
lab

## Title
Skills Assessment — Using Web Proxies

## Description
Internal pentest skills assessment covering response manipulation via ZAP Replacer, multi-layer cookie decoding (ASCII hex → Base64), Burp Intruder fuzzing with chained payload processing, and proxying Metasploit traffic through Burp/ZAP for request inspection.

## Tags
zap, burp-suite, intruder, cookie-decoding, proxy, metasploit

## Commands
- `disabled>` → `>` (ZAP Replacer: Match Type = Response Body String)
- Encode/Decode/Hash → ASCII Hex Decode → Base64 Decode
- `set PROXIES HTTP:127.0.0.1:8080`
- `use auxiliary/scanner/http/coldfusion_locale_traversal`
- `set RHOSTS <TARGET_IP>`
- `set RPORT <TARGET_PORT>`

## What This Section Covers
Four scenarios testing core web proxy skills: manipulating disabled HTML elements via response modification, decoding multi-layered cookies, fuzzing partial MD5 hashes with chained encoding in Burp Intruder, and routing Metasploit traffic through a proxy to inspect requests.

## Methodology

### Q1 — Enable Disabled Button (/lucky.php)
1. Set FoxyProxy to route through ZAP (port 8080) and navigate to `/lucky.php`
2. Open ZAP Replacer (`Ctrl+R`) → Add rule: Match Type = `Response Body String`, Match String = `disabled>`, Replacement = `>`
3. Right-click the GET request → `Open/Resend with Request Editor` → Send → confirm `disabled` is stripped from the response
4. Right-click response → `Open URL in System Browser` → hard refresh with `Ctrl+Shift+R` if cached
5. Click the button ~8 times until the flag appears

### Q2 — Decode Multi-Encoded Cookie (/admin.php)
1. Navigate to `/admin.php` with proxy active and find the `Cookie:` header in ZAP
2. Select the cookie value → right-click → `Encode/Decode/Hash...`
3. The raw value is all hex chars (`0-9`, `a-f`) → **ASCII Hex encoded**
4. ASCII Hex Decode reveals a Base64 string → **Base64 encoded**
5. Base64 Decode reveals the 31-character MD5 hash: `3dac93b8cd250aa8c1a36fffc79a17a`

### Q3 — Fuzz Missing MD5 Character (Burp Intruder)
1. Switch FoxyProxy to Burp (8080), navigate to `/admin.php`, send the request to **Intruder**
2. Clear all markers → replace the cookie value with the 31-char hash → wrap it in `§§` markers: `cookie=§3dac93b8cd250aa8c1a36fffc79a17a§`
3. Payloads tab → Load `/opt/useful/SecLists/Fuzzing/alphanum-case.txt`
4. Add three **Payload Processing** rules in order:
   - **Add prefix** → `3dac93b8cd250aa8c1a36fffc79a17a`
   - **Base64-encode**
   - **Encode → ASCII hex**
5. Start attack → sort by **Length** → responses with size **1248** contain the flag

### Q4 — Proxy Metasploit Traffic
1. Launch `msfconsole -q`
2. `use auxiliary/scanner/http/coldfusion_locale_traversal`
3. `set PROXIES HTTP:127.0.0.1:8080` → `set RHOSTS <TARGET_IP>` → `set RPORT <TARGET_PORT>`
4. Ensure Burp/ZAP is intercepting → `run`
5. Inspect the intercepted request path to find the directory name

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 | HTB{d154bl3d_bu770n5_w0n7_570p_m3} | ZAP Replacer strips `disabled` from button → click ~8 times |
| Q2 | 3dac93b8cd250aa8c1a36fffc79a17a | ASCII Hex Decode → Base64 Decode of /admin.php cookie |
| Q3 | HTB{burp_1n7rud3r_n1nj4!} | Burp Intruder fuzzing last MD5 char with chained encoding |
| Q4 | CFIDE | Proxied Metasploit request shows /CFIDE/administrator/.. |

## Key Takeaways
- **Encoding recognition by character set:** all lowercase hex = ASCII hex; mixed case + `=` padding = Base64. Peel layers until you hit plaintext, note the order, reverse it to decode, replicate it to encode.
- **Payload Processing order matters:** rules execute top-to-bottom. For re-encoding, apply them in the same order the server originally encoded (prefix → Base64 → ASCII hex).
- **ZAP Replacer vs Burp Match & Replace:** both do response body modification; ZAP Replacer may require specifying the target URL explicitly.
- **Burp Community Intruder is throttled:** for real engagements use Turbo Intruder extension, custom Python scripts, or ffuf where encoding chains permit.
- **Any tool can be proxied through Burp/ZAP** by setting its proxy config (Metasploit uses `set PROXIES HTTP:127.0.0.1:8080`), useful for inspecting and replaying tool-generated requests.
- **Cache can mask response changes** — always hard-refresh (`Ctrl+Shift+R`) after modifying responses via proxy rules.

## Gotchas
- If the button on `/lucky.php` still appears disabled after Replacer is set, it's likely a cached page — force refresh with `Ctrl+Shift+R`
- ZAP lacks a built-in ASCII hex fuzzer processor, making Q3 much harder in ZAP — use Burp Intruder for chained encoding tasks
- In Burp Intruder, the payload markers wrap the entire hash value because the prefix processing rule re-adds it — the marker position defines what gets replaced, the prefix rebuilds the hash + appended fuzz character


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-proxies]]  
← [[14-extensions]]
<!-- AUTO-LINKS-END -->
