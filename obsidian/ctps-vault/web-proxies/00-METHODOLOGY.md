# NOTE — Using Web Proxies Methodology (Exam Playbook)

## ID
708

## Module
Using Web Proxies

## Kind
methodology

## Title
Using Web Proxies — Burp / ZAP Operations Playbook

## Description
Exam-ready retrieval tool for driving Burp Suite and ZAP under time pressure: get traffic flowing → intercept/modify requests → modify responses → automate match-and-replace → repeat → encode/decode → fuzz → scan → proxy CLI tools. Operation-indexed (find the task, copy the steps), commands drawn from this vault's own notes.

## Tags
methodology, web-proxies, burp, burp-suite, zap, owasp-zap, exam, cheatsheet, decision-tree, intercept, repeater, intruder, fuzzer, match-and-replace, foxyproxy, ca-certificate, encoding, decoding, base64, ascii-hex, proxychains, metasploit-proxy, command-injection, panic

---

## TL;DR — The 9 Operations

1. **Setup** — FoxyProxy → 127.0.0.1:8080, install the CA cert. No cert = HTTPS won't decrypt.
2. **Intercept request** — pause before server, edit param/header, forward. Bypasses front-end JS validation.
3. **Intercept response** — edit HTML before render. Bypasses `disabled` / `maxlength` / `type=number`.
4. **Automatic modification** — match-and-replace rules; persistent, no manual intercept.
5. **Repeat** — Repeater / Request Editor; iterate payloads without re-intercepting.
6. **Encode/Decode** — peel layers (ASCII-hex → Base64 → plaintext), reverse order to re-encode.
7. **Fuzz** — Burp Intruder (throttled free) **or** ZAP Fuzzer (no throttle — prefer for real fuzzing).
8. **Scan** — Burp Scanner (Pro only) / ZAP Spider + Active Scan (free) → triage High alerts.
9. **Proxy other tools** — `proxychains -q` or Metasploit `set PROXIES` to inspect/replay tool traffic.

> **Golden rule:** the proxy is your eyes — if you can't *see* the raw request/response, you can't test it. Before debugging anything, confirm traffic is actually landing in Proxy History. Empty history = setup is broken, not the app.

> **Tool fork (the exam-time decision):** free **Burp Intruder is rate-limited to ~1 req/sec** — useless for anything but tiny/targeted fuzzes. For real fuzzing use **ZAP Fuzzer (no throttle)**. *But* ZAP has **no ASCII-hex payload processor** — chained-encoding fuzzes (prefix → Base64 → ASCII-hex) **must** be done in Burp Intruder. See Gotcha #9.

---

## Operation 1 — Setup (Get Traffic Flowing)

**Goal:** browser → proxy → target, with HTTPS decrypting cleanly.

| Need | Fastest path |
|---|---|
| Zero-config quick test | Burp: **Proxy → Intercept → Open Browser** (Chromium, cert pre-installed). ZAP: Firefox icon, or **Manual Explore → Launch Browser** |
| Real browser (longer test) | FoxyProxy → `127.0.0.1:8080` (already preset in PwnBox/Pwnbox) + import CA cert |
| ZAP without FoxyProxy | Firefox → Settings → search "proxy" → Manual proxy `127.0.0.1:8080`, tick "Also use for HTTPS", clear "No proxy for" |

```bash
# Verify the proxy is actually listening + reachable
ss -tlnp | grep 8080
curl -x http://127.0.0.1:8080 http://<TARGET_IP>:<PORT>/   # should appear in Proxy History
```

CA cert install (one-time per browser profile):
- **Burp:** proxy on → browse `http://burp` → **CA Certificate** → download.
- **ZAP:** **Tools → Options → Network → Server Certificates → Save**.
- **Firefox:** `about:preferences#privacy` → View Certificates → **Authorities → Import** → tick **"Trust this CA to identify websites"**.

**Output checkpoint:** a manual `curl -x` and a browser request both show up in Proxy History; HTTPS pages load with no cert warning.

---

## Operation 2 — Intercept & Modify a Request

**Trigger:** front-end JS restricts input (number-only field, maxlength) and you suspect the back end doesn't re-validate.

1. FoxyProxy → **Burp (8080)**; **Proxy → Intercept** = **on**.
2. Trigger the action in the browser; click **Forward** through noise (Firefox telemetry) until you see the target request (e.g. `POST /ping`).
3. `Ctrl+R` → send to **Repeater** (so you can iterate — don't keep re-intercepting).
4. Edit the param (`ip=1` → `ip=;cat flag.txt`), highlight payload → **`Ctrl+U`** to URL-encode → `ip=%3bcat+flag.txt`.
5. **Send**; read response inline.

ZAP equivalent: HUD lets you intercept/edit in-browser; or History → right-click → **Open/Resend with Request Editor**.

**Output checkpoint:** modified param reaches the server (response reflects the injection, not a front-end rejection).

---

## Operation 3 — Intercept & Modify a Response

**Trigger:** a button is `disabled`, a field is `type="number"` / `maxlength="3"`, or an element is hidden — and the restriction is client-side only.

1. **Burp → Proxy → Proxy Settings → Response Interception Rules → tick "Intercept Server Responses".**
2. Request interception on → hard-reload (`Ctrl+Shift+R`) → **Forward** the request; the response is now caught.
3. Edit HTML: `type="number"`→`type="text"`, `maxlength="3"`→`maxlength="100"`, strip `disabled`.
4. **Forward** the modified response to the browser.
5. ZAP: intercept request, **Step** to catch the response; or HUD **Show/Enable** (light-bulb) for one-click un-disable/unhide.
6. Lazy automation: Burp **Proxy Settings → Response Modification Rules → "Unhide hidden form fields"**.

**Output checkpoint:** the previously-blocked control is usable in the rendered page (hard-refresh if it still looks blocked — cache).

---

## Operation 4 — Automatic Modification (Match & Replace)

**Trigger:** you need the same edit applied to *every* request/response (User-Agent spoof, permanent client-side bypass, payload on every submit).

```text
# Burp: Proxy → Proxy settings → HTTP match and replace rules → Add
Type: Request header   Match: ^User-Agent.*$   Replace: User-Agent: HackTheBox Agent 1.0   Regex: True
Type: Response body    Match: type="number"    Replace: type="text"                        Regex: False
Type: Response body    Match: maxlength="3"     Replace: maxlength="100"

# ZAP: Ctrl+R → Replacer → Add  (or Options → Replacer)
Match Type: Response Body String   Match: disabled>     Replace: >
Match Type: Request Body String    Match: ip            Replace: ip;ls;
Match Type: Request Header String  Match: User-Agent    Replace: HackTheBox Agent 1.0
```

Force-refresh (`Ctrl+Shift+R`) — changes now persist across reloads, no manual intercept.

**Output checkpoint:** the rule fires automatically on a fresh page load (verify in Repeater/History that the substitution happened).

---

## Operation 5 — Repeat Requests (Iterate Without Re-Intercepting)

**Trigger:** manual enumeration (command-injection sweep, param tampering) where you'd otherwise do 5–6 steps per payload.

1. Send a normal request through the proxy (Burp **Proxy > HTTP History** / ZAP History pane).
2. Burp `Ctrl+R` → Repeater (`Ctrl+Shift+R` to switch tab). ZAP: right-click → **Open/Resend with Request Editor**.
3. Edit param in place, **Send**, read response, repeat with next payload.

```text
ip=127.0.0.1;ls+/
ip=127.0.0.1;find+/+-name+"flag*"
ip=127.0.0.1;cat+<FLAG_PATH>
```

Burp keeps Original vs Edited request; ZAP shows only the final. Right-click → **Change Request Method** flips GET/POST without rewriting.

**Output checkpoint:** a working payload identified by iterating in one pane (no browser round-trips).

---

## Operation 6 — Encode / Decode

**Trigger:** request contains URL-encoded params, a Base64/hex cookie, or you must re-encode a tampered value the way the server expects.

- **Burp:** select text → right-click → **Convert Selection → URL → URL-encode key characters**; or `Ctrl+U` (encode as you type). **Decoder** tab for Base64/HTML/Unicode/ASCII-hex. Inspector → right-click → **Decode as…**.
- **ZAP:** **`Ctrl+E`** → Encode/Decode/Hash; paste value → it auto-tries multiple decoders in parallel. Encode tab to re-encode. **Add New Tab** for a custom encoder set.

**Layered-cookie recipe (Skills Assessment Q2):**
```
raw value = all 0-9a-f only      → ASCII Hex  → decode
result    = mixed case + '=' pad → Base64     → decode
result    = 31-char hex          → MD5 hash (plaintext target)
```
**Recognition rule:** all-lowercase-hex = ASCII hex; mixed-case + `=` padding = Base64. Peel until plaintext, note the order, **reverse it to decode, replay it to re-encode.**

**Output checkpoint:** value decoded to plaintext and the exact encode order recorded for re-encoding.

---

## Operation 7 — Fuzzing

**Decide tool FIRST** (see Tool Fork): chained encoding → **Burp Intruder**; everything else / speed → **ZAP Fuzzer**.

### 7.A — Burp Intruder (throttled ~1 req/s on free)

1. History → request → **`Ctrl+I`** (`Ctrl+Shift+I` to switch tab).
2. **Positions:** `Clear §`, place markers — e.g. `GET /admin/§value§.html` (Sniper = 1 position/1 list; Cluster Bomb = multi-position).
3. **Payloads:** `Simple List` → `Load` → `/opt/useful/seclists/Discovery/Web-Content/common.txt` (use **Runtime file** for huge lists).
4. **Payload Processing:** `Skip if matches regex` `^\..*$` (skip dotfiles); or prefix/Base64/ASCII-hex chains (order = top-to-bottom).
5. **Settings → Grep - Match:** `Clear`, add `200 OK`, untick **Exclude HTTP Headers**.
6. `Start Attack` → sort by `200 OK` / `Status` / `Length` column → checkmark = hit.

### 7.B — ZAP Fuzzer (no throttle — real fuzzing)

1. History → request → **Attack > Fuzz**.
2. Select the value → `Add` = **Fuzz Location**.
3. Payloads `Add` → `File` (custom path) or `File Fuzzers` (built-in / FuzzDB after Marketplace install).
4. **Processors** → `MD5 Hash` / `URL Encode` / Prefix-Postfix → `Generate Preview` to verify.
5. **Options → Concurrent Scanning Threads = 20**.
6. `Start Fuzzer` → sort by **Size Resp. Body** / Response Code / RTT → the outlier is the hit.

**MD5-cookie fuzz gotcha:** first visit only gets `Set-Cookie` in the *response*; **refresh (`F5`)** so the *second* request carries the cookie — fuzz that one.

**Output checkpoint:** one result stands out (different status/size/time) → open it for the flag.

---

## Operation 8 — Scanning

| Tool | Availability | Flow |
|---|---|---|
| **Burp Scanner** | **Pro/Enterprise ONLY** (Community can't scan) | Add to scope → Dashboard → New Scan → Crawl / Crawl+Audit → filter Severity:High + Confidence:Certain/Firm → Issue → Report |
| **ZAP Scanner** | **Free** | History → Attack > **Spider** (+ **Ajax Spider** for JS links) → **Active Scan** → Alerts by severity → click High → replay |

ZAP: Passive Scanner runs automatically during spidering (missing headers, DOM XSS). Always open **High alerts first** (RCE/SQLi → direct compromise); the alert shows the exact request + payload ZAP used — replay it. `Report > Generate HTML Report` to export.

**Output checkpoint:** site tree built, High-severity issue confirmed and reproduced manually.

---

## Operation 9 — Proxy CLI Tools / Metasploit

**Trigger:** need to see/replay what a CLI tool or MSF module actually sends.

```bash
# proxychains — universal, any CLI tool
sudo nano /etc/proxychains.conf      # comment 'socks4 127.0.0.1 9050'; add: http 127.0.0.1 8080
proxychains -q curl http://<TARGET_IP>:<PORT>     # -q = quiet (suppress connection noise)

# Metasploit — built-in, no proxychains needed
msfconsole -q
use auxiliary/scanner/http/http_put
set PROXIES HTTP:127.0.0.1:8080
set RHOSTS <TARGET_IP>
set RPORT  <PORT>
run                                   # every HTTP request now visible in Proxy History
```

Inspect in Proxy History → send to Repeater → replay/modify with manual precision.

**Output checkpoint:** the tool's requests appear in Proxy History; you can read/replay them.

---

## Decision Tree (Under Exam Pressure)

```
I need to...
│
├── see traffic but History is EMPTY
│   ├── ss -tlnp | grep 8080  → proxy not listening? relaunch
│   ├── curl -x http://127.0.0.1:8080 http://TARGET/  → reaches History? browser is the problem
│   ├── Burp on 8080 → ZAP silently moved to 8081 → match FoxyProxy/Firefox to actual port
│   └── still nothing → use the tool's built-in browser (Open Browser / Manual Explore)
│
├── HTTPS shows cert warnings everywhere
│   └── CA cert not imported / "Trust to identify websites" unticked → Operation 1
│
├── bypass a front-end restriction
│   ├── input filtered BEFORE submit (JS) → intercept REQUEST, edit param (Op 2)
│   ├── button disabled / maxlength / type=number / hidden field → intercept RESPONSE (Op 3)
│   └── need it on every page automatically → Match & Replace / Replacer (Op 4)
│
├── test many payloads on ONE param
│   └── Repeater / Request Editor (Op 5) — never re-intercept per payload
│
├── value is encoded (cookie/param unreadable)
│   └── Decoder / Ctrl+E (Op 6): hex-only=ASCII hex, mixed+'='=Base64; peel, note order, reverse
│
├── brute/fuzz dirs, files, params, cookies
│   ├── chained encoding (prefix→B64→hex) → Burp Intruder ONLY (ZAP can't ASCII-hex) (Op 7.A)
│   ├── speed matters / big list → ZAP Fuzzer (no throttle) (Op 7.B)
│   └── tiny targeted list, Burp open already → Burp Intruder (slow but fine)
│
├── automated vuln discovery
│   ├── have Burp Pro → Burp Scanner (Op 8)
│   └── Community only → ZAP Spider → Active Scan (free, no limit) (Op 8)
│
├── inspect what a CLI tool / MSF module sends
│   └── proxychains -q  OR  set PROXIES HTTP:127.0.0.1:8080 (Op 9)
│
└── STUCK > 15 min
    ├── grep ATTACK-PATHS.md §3c for the symptom
    ├── confirm traffic in History at all (Golden rule) before blaming the app
    ├── hard-refresh Ctrl+Shift+R (response edits get cached)
    └── front-end rejecting? you're editing too late — intercept earlier / use Match&Replace
```

---

## Signal → Counter-Move Reference

| You see / observe | Likely cause | Counter-move |
|---|---|---|
| Proxy History stays empty | Browser not proxying / wrong port | `ss -tlnp \| grep 8080`; `curl -x` test; use tool's built-in browser |
| ZAP captures nothing, Burp is running | Burp took 8080 → ZAP auto-moved to **8081** | Tools→Options→Network→Local Servers; point FoxyProxy/proxychains at the real port |
| Every HTTPS site = cert warning | CA cert not imported, or "Trust to identify websites" unticked | Re-import in **Authorities**, tick the box |
| `http://burp` cert page won't load | Burp not running / proxy not enabled in browser | Enable proxy + Burp first, then browse `http://burp` |
| Browser hangs, pages "broken" | Intercept left ON (or proxy on, Burp/ZAP closed) | Toggle Intercept off / FoxyProxy off |
| Injection fails, server misparses body | Payload not URL-encoded | Highlight → `Ctrl+U` (Burp); `+`=space, `%XX`=special |
| Endless Firefox requests before your target | Telemetry/update noise | Click **Forward** repeatedly until the target verb/path appears |
| Modified response still shows old/disabled UI | Browser cached the original | Hard refresh **`Ctrl+Shift+R`** / `Ctrl+F5` |
| Match&Replace literal rule corrupts traffic | Match too broad (plain `User-Agent`) | Anchor with regex `^User-Agent.*$` |
| Strange app behaviour, can't explain it | Forgotten enabled Match&Replace rule | Audit Proxy settings → disable rules |
| Free Burp Intruder crawling at 1 req/s | Community throttle (by design) | Switch to **ZAP Fuzzer**, or Turbo Intruder ext / ffuf |
| ZAP fuzz can't do chained hex encoding | ZAP has **no ASCII-hex processor** | Do the chained-encoding fuzz in **Burp Intruder** |
| MD5-cookie fuzz returns nothing | First request had no cookie (only `Set-Cookie` in response) | **`F5`** once, fuzz the *second* request that sends the cookie |
| Burp "Scan" option greyed out | Community edition — no scanner | Use **ZAP Spider + Active Scan** (free) |
| Active scan finds nothing past start | Scope not set / killed early | Add to scope; let Active Scan finish (it's slow by design) |
| ZAP HUD buttons unresponsive | HUD/browser version glitch | Use main ZAP UI (right-click in History/Sites) |
| Cmd-injection separator rejected | Server parses input differently | Try both `;cmd;` and `&cmd&` forms |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# === Verify proxy plumbing ===
ss -tlnp | grep 8080
curl -x http://127.0.0.1:8080 http://<TARGET_IP>:<PORT>/

# === Proxy a CLI tool ===
sudo nano /etc/proxychains.conf            # add: http 127.0.0.1 8080
proxychains -q curl http://<TARGET_IP>:<PORT>/

# === Proxy Metasploit ===
msfconsole -q
use auxiliary/scanner/http/http_put
set PROXIES HTTP:127.0.0.1:8080
set RHOSTS <TARGET_IP>; set RPORT <PORT>; run

# === Command-injection payloads to iterate in Repeater ===
ip=%3bcat+flag.txt                          # ;cat flag.txt  (URL-encoded)
ip=127.0.0.1;ls+/
ip=127.0.0.1;find+/+-name+"flag*"
ip=127.0.0.1;cat+/flag.txt;
ip=127.0.0.1&cat /flag.txt&                 # & form if ; rejected
```

```text
# === Burp Match & Replace (Proxy → Proxy settings → HTTP match and replace) ===
Request header  | ^User-Agent.*$ | User-Agent: HackTheBox Agent 1.0 | Regex
Response body   | type="number"  | type="text"
Response body   | maxlength="3"  | maxlength="100"

# === ZAP Replacer (Ctrl+R → Replacer → Add) ===
Response Body String | disabled> | >
Request Body String  | ip        | ip;ls;

# === Burp Intruder dir-fuzz ===
Positions: GET /admin/§value§.html
Payloads : Simple List → Load common.txt
Process  : Skip if matches regex  ^\..*$
Settings : Grep-Match → 200 OK , untick Exclude HTTP Headers

# === Burp Intruder chained-encoding cookie fuzz (Skills Assessment Q3) ===
Marker   : cookie=§<31-char-md5>§
Payloads : alphanum-case.txt
Process  : 1) Add prefix <31-char-md5>  2) Base64-encode  3) Encode → ASCII hex
Sort     : Length  (1248 = flag)

# === ZAP Fuzzer MD5-cookie fuzz ===
F5 to re-send cookie → right-click req → Attack > Fuzz
Fuzz Location = cookie value | Payload File = top-usernames-shortlist.txt
Processor = MD5 Hash | Threads = 20 | sort by Size Resp. Body

# === Layered decode (Ctrl+E / Burp Decoder) ===
hex-only(0-9a-f) → ASCII Hex Decode → Base64 Decode → plaintext   # reverse to re-encode
```

---

## Quick Reference — Capability by Tool

| Function | Burp Suite | OWASP ZAP |
|---|---|---|
| Intercept request/response | Proxy → Intercept | HUD / Break, **Step** for response |
| Repeat | Repeater (`Ctrl+R`) | Request Editor (right-click → Open/Resend) |
| Auto modify | HTTP match and replace rules | Replacer (`Ctrl+R` → Replacer) |
| Encode/Decode | Decoder tab / `Ctrl+U` / Inspector | Encode/Decode/Hash (`Ctrl+E`) — **no ASCII-hex processor** |
| Fuzz | Intruder (`Ctrl+I`) — **~1 req/s free** | Fuzzer (Attack > Fuzz) — **no throttle** |
| Payload processors | Prefix/Suffix/Regex/Base64/**ASCII-hex**/skip | MD5/SHA/URL-encode/Prefix-Postfix (no ASCII-hex) |
| Scanner | **Pro only** (Crawl+Audit) | **Free** Spider + Ajax Spider + Active/Passive |
| Out-of-band | Collaborator (Pro) | external OAST |
| Extensions | BApp Store (some Pro-only, may need Jython) | Marketplace (all free; FuzzDB for wordlists) |
| Built-in browser | Open Browser (Chromium) | Manual Explore → Launch Browser |

Default port **8080** for both → if both run, ZAP slides to **8081**.

---

## Top Gotchas (Things That Will Burn You)

1. **No CA cert = no HTTPS.** Forgetting the cert import (and ticking "Trust this CA to identify websites") is the #1 setup failure — every HTTPS page warns or blocks.
2. **Empty Proxy History means setup, not the app.** Verify with `ss -tlnp | grep 8080` + `curl -x` before touching the target. Golden rule.
3. **Burp + ZAP port collision.** Burp grabs 8080, ZAP silently moves to 8081 — FoxyProxy/proxychains still point at 8080 → nothing captured. Pick one port, make everything match.
4. **Intercept left ON makes the whole browser hang.** "App is broken" is almost always your own intercept toggle or FoxyProxy still on with the proxy closed.
5. **URL-encode injection payloads (`Ctrl+U`).** Un-encoded `;` `&` `#` corrupt the POST body — injection silently fails.
6. **Response edits get cached.** After modifying a response (manual or Match&Replace), hard-refresh `Ctrl+Shift+R` or it looks like nothing changed.
7. **Free Burp Intruder is ~1 req/sec by design.** Don't burn exam time waiting — use ZAP Fuzzer (no throttle) for anything non-trivial.
8. **MD5-cookie fuzz needs an `F5` first.** First request only receives `Set-Cookie` in the *response*; the cookie isn't *sent* until the second request. Fuzz the second one.
9. **ZAP has no ASCII-hex payload processor.** Chained prefix→Base64→ASCII-hex fuzzes (Skills Assessment Q3) are Burp-Intruder-only. Don't waste time fighting ZAP for this.
10. **Burp Scanner is Pro/Enterprise only.** Community can't passive- or active-scan. On the exam (Community) use **ZAP Spider + Active Scan** — also faster (no throttle).
11. **Crawler ≠ ffuf.** Burp/ZAP spiders follow links only; they do **not** brute-force directories. Undiscovered endpoints stay untested — pair with ffuf/Intruder.
12. **Active Scan is intrusive and slow.** It can trigger destructive actions (logout, CRUD) — set scope tightly; don't kill it early or you miss findings.
13. **Match&Replace too broad corrupts traffic.** Plain literal `User-Agent` in body matches everywhere — anchor regex `^User-Agent.*$`.
14. **Payload Processing order is top-to-bottom.** To re-encode, apply rules in the *same order the server originally encoded* (prefix → Base64 → ASCII-hex), not reversed.
15. **Encoding recognition by charset:** all-lowercase-hex → ASCII hex; mixed-case + `=` padding → Base64. Peel to plaintext, record order, reverse to decode / replay to encode.
16. **Match&Replace / response rules don't survive a proxy restart** unless the Burp project (Pro) or ZAP session is saved.

---

## Related Vault Notes

- `[[01-intro]]` — proxy concept, Burp vs ZAP comparison table
- `[[02-installation]]` — install/launch, temporary vs persistent projects
- `[[03-proxy-setup]]` — FoxyProxy + CA certificate (the #1 gotcha)
- `[[04-intercepting-requests]]` — request intercept → command injection
- `[[05-intercepting-responses]]` — HTML edit, ZAP HUD Show/Enable
- `[[06-automatic-modification]]` — Match & Replace / Replacer
- `[[06-zap-replacer-notes]]` — Replacer exercise Q&A
- `[[07-repeating-requests]]` — Repeater / Request Editor iteration
- `[[08-encoding-decoding]]` — Decoder / Ctrl+E, layered decode
- `[[09-proxying-tools]]` — proxychains / Metasploit PROXIES
- `[[10-burp-intruder]]` — positions, payloads, processing, grep-match
- `[[11-zap-fuzzer]]` — no-throttle fuzzing, processors, MD5-cookie lab
- `[[12-burp-scanner]]` — Pro scanner: crawl/passive/active/report
- `[[13-zap-scanner]]` — Spider / Ajax Spider / Active Scan (free)
- `[[14-extensions]]` — BApp Store / ZAP Marketplace, FuzzDB
- `[[skills-assessment]]` — Replacer + layered decode + chained-encoding Intruder + MSF proxy

External cross-vault:
- Command injection targets: `[[../command-injetions/00-METHODOLOGY]]`
- Content discovery (CLI fuzzer alternative): `[[../ffuf/00-METHODOLOGY]]`
- Web app attacks the proxy enables: `[[../web-attacks/00-METHODOLOGY]]`, `[[../xss/00-METHODOLOGY]]`, `[[../sql-injection-fundamentals/00-METHODOLOGY]]`
- Triage by symptom: `[[../ATTACK-PATHS]]` §3c (web proxy operations)
- Index: `[[../SEARCH]]`
- Exam pitfalls: `[[../EXAM-WARNINGS]]`
