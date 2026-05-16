# NOTE — Attacking Web Applications with Ffuf Methodology (Exam Playbook)

## ID
716

## Module
Ffuf

## Kind
methodology

## Title
Attacking Web Applications with Ffuf — Full Web-Fuzzing Methodology

## Description
End-to-end exam-ready playbook for content discovery with ffuf: directory → page/extension → recursion → host discovery (sub-domain vs vhost) → filter calibration → parameter (GET/POST) → value → flag. Decision-tree first; every command drawn from this vault's own ffuf notes.

## Tags
methodology, ffuf, fuzzing, web-fuzzing, exam, cheatsheet, decision-tree, directory-fuzzing, page-fuzzing, extension-fuzzing, recursion, subdomain-fuzzing, vhost-fuzzing, host-header, parameter-fuzzing, value-fuzzing, filtering, calibration, baseline, every-result-200, errors-4997, fs-filter, ac-autocalibrate, FUZZ, seclists, etc-hosts

---

## TL;DR — The 7-Phase Flow

1. **Directory fuzz** — `FUZZ` in the URL path, find top-level dirs.
2. **Page & extension fuzz** — two-stage: confirm extension on `index`, then fuzz filenames.
3. **Recursive fuzz** — collapse "find dir → cd → fuzz again" into one `-recursion` run.
4. **Host discovery** — sub-domain fuzz (public DNS) → fails on labs → **vhost fuzz** (`Host:` header).
5. **Filter calibration** — when every miss returns the same 200, `-fs <baseline>` or `-ac`. *Make-or-break step.*
6. **Parameter fuzz** — `?FUZZ=key` (GET) → `-X POST -d 'FUZZ=key'` (POST). Re-baseline each.
7. **Value fuzz** — `FUZZ` in the value slot with a custom/seq wordlist → curl+grep the flag.

> **Golden rule:** ffuf reports *hit vs miss*, not page content. Every confirmed hit must be re-issued with `curl -s ... | grep` to extract the actual data/flag. And: **the filter baseline changes at every stage** — vhost ≠ extension ≠ parameter ≠ value. Re-calibrate `-fs` each time.

> **OPSEC / speed fork:** default 40 threads scans ~87k entries in <10s — *do not* crank `-t 200` on production. On isolated lab/exam targets `-t 100` is fine to push long chains. If every result is `200 OK same size`, **don't keep scrolling** — stop and calibrate the filter (Phase 5) before wasting more requests.

---

## Phase 1 — Directory Fuzzing

**Goal:** enumerate top-level directories the server doesn't link to.

| Trigger / Precondition | Action |
|---|---|
| Web port open, page reachable, no site map | Start here |
| You have a domain/vhost confirmed | Fuzz dirs on it |

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
  -u http://SERVER_IP:PORT/FUZZ -ic
# clean/pipeable:
ffuf -s -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
  -u 'http://STMIP:STMPO/FUZZ'
```

- `-w <list>:FUZZ` binds the keyword; `-u` places it. Default match codes: `200,204,301,302,307,401,403`.
- `-ic` skips the wordlist's copyright comment lines (otherwise fuzzed as candidates).
- A `301` is a **real directory** (redirect to trailing-slash form). An empty `200 Size:0` still exists — drill in with Phase 2.

**Output checkpoint:** after this you have a list of directories → pick one and fuzz files inside it.

Detail: `[[03-directory-fuzzing]]`, theory `[[02-web-fuzzing]]`.

---

## Phase 2 — Page & Extension Fuzzing

**Goal:** find hidden files inside a directory. Two stages — never fuzz name+ext at once.

| Trigger / Precondition | Action |
|---|---|
| Found a directory, want files | Stage 1: confirm extension |
| Know the extension | Stage 2: fuzz filenames |

```bash
# Stage 1 — confirm extension against the universal 'index' anchor
ffuf -w /opt/useful/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ \
  -u http://SERVER_IP:PORT/blog/indexFUZZ

# Stage 2 — fix the extension, fuzz the filename
ffuf -s -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
  -u 'http://STMIP:STMPO/blog/FUZZ.php'
```

- `web-extensions.txt` **already includes the leading dot** — write `indexFUZZ`, not `index.FUZZ` (or you get `index..php`).
- `200 Size:0` on `.php` = the language is parsed/enabled (empty body). `403` on the ext also confirms it's parsed.
- Visit **non-zero-size** hits — those have real content. Treat a `.phps` 200 as a source-disclosure jackpot.

**Output checkpoint:** a real page (e.g. `/blog/home.php`) → visit/curl it for content or the flag.

Detail: `[[04-page-fuzzing]]`.

---

## Phase 3 — Recursive Fuzzing (the speed-up)

**Goal:** one command instead of "fuzz → find dir → cd → fuzz → repeat".

| Trigger / Precondition | Action |
|---|---|
| Multi-level site, want full tree | Add `-recursion` |
| Need files + dirs in one pass | Add `-e .php,...` |

```bash
ffuf -s -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
  -u 'http://STMIP:STMPO/FUZZ' -recursion -recursion-depth 1 -e '.php' -v
```

- `-recursion-depth 1` = one level below start. **Always start at depth 1** — depth 2 on a wide site runs for hours.
- Recursion math: `wordlist × extensions × depth`. A 4-ext list at depth 2 = 8× the requests.
- `-v` is mandatory with recursion — without it identical filenames across subdirs are ambiguous.
- ffuf does **not** retro-fuzz parents; depth is fixed at start.

**Exam speed trick:** the instant ffuf prints `Adding a new job to the queue: .../courses/FUZZ`, kill it and restart targeting that path directly — saves minutes.

**Output checkpoint:** full discovered tree with files → hand interesting pages to Phase 6.

Detail: `[[05-recursive-fuzzing]]`.

---

## Phase 4 — Host Discovery: Sub-domain vs Vhost

**Goal:** find other sites on the target. Pick the branch by whether public DNS exists.

| Symptom | Branch |
|---|---|
| Public internet domain (e.g. `inlanefreight.com`) | 4.A Sub-domain (DNS) |
| Lab `*.htb` / no public DNS / `Errors: 4997` | 4.B Vhost (Host header) — *always works* |

### 4.A — Sub-domain fuzzing (public DNS only)

```bash
ffuf -s -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
  -u 'http://FUZZ.inlanefreight.com/'
# escalate coverage: swap to subdomains-top1million-110000.txt
```

A wall of `Errors:` / `0 hits` against a lab domain = **switch to 4.B**.

### 4.B — Vhost fuzzing (no DNS needed — `Host:` header)

```bash
# /etc/hosts FIRST so the IP resolves
sudo sh -c 'echo "SERVER_IP  academy.htb" >> /etc/hosts'

ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ \
  -u http://academy.htb:PORT/ -H 'Host: FUZZ.academy.htb' -fs <baseline>
```

- Request goes **to the IP** (resolved via `/etc/hosts`); `FUZZ` varies the `Host:` header.
- Non-existent vhost → server returns the **default vhost page** → every miss is `200 OK, identical size`. **You cannot read this without Phase 5 filtering.**
- After a vhost is found, add it to `/etc/hosts` (single multi-host line) before visiting.

**Output checkpoint:** a list of vhosts/sub-domains. Each is a **new target** — restart the chain (dirs/params) on each.

Detail: `[[06-dns-records]]`, `[[07-sub-domain-fuzzing]]`, `[[08-vhost-fuzzing]]`.

---

## Phase 5 — Filter Calibration (make-or-break)

**Goal:** strip the baseline-response noise that floods vhost / parameter / value scans. ffuf's only default filter is the status code; when every miss is `200 same size`, you need a second axis.

| Trigger / Precondition | Action |
|---|---|
| Every result `200`, identical `Size:` | `-fs <baseline>` |
| Don't want to read the baseline manually | `-ac` (auto-calibrate) |
| Baseline page is dynamic (timestamps/random nav) | `-fl` (lines) or `-fw` (words) — more stable than size |
| A hit accidentally matches baseline size | Multi-axis: `-fs X -fw Y` |

```bash
# 1. Run once unfiltered, read the Size: column = baseline
# 2. Re-run filtering it out
ffuf -w wordlist:FUZZ -u http://academy.htb:PORT/ -H 'Host: FUZZ.academy.htb' -fs 986
# OR let ffuf calibrate from junk requests:
ffuf -w wordlist:FUZZ -u http://academy.htb:PORT/ -H 'Host: FUZZ.academy.htb' -ac
```

| Match (keep) | Filter (drop) |
|---|---|
| `-mc` codes / `-ms` size / `-mw` words / `-ml` lines / `-mr` regex | `-fc` codes / `-fs` size / `-fw` words / `-fl` lines / `-fr` regex |

- `-mr "You don't have access!"` matches a known protected-page string directly — skips the find-dir-then-fuzz two-step on assessments.
- `-ac` is the no-brain default but **re-calibrates every run** — on dynamic pages it can drift and miss hits. Compare runs vs manual `-fs`.

**Output checkpoint:** clean output, only real divergent hits. This filter is now your tool for Phases 6–7.

Detail: `[[09-filtering-results]]`.

---

## Phase 6 — Parameter Fuzzing (GET → POST)

**Goal:** discover undocumented parameters (debug flags, legacy/internal names) — they're less hardened because nobody tests them.

| Symptom | Branch |
|---|---|
| Endpoint with query string / "no access" page | 6.A GET |
| GET yields nothing, or param is hidden | 6.B POST |

### 6.A — GET parameters

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u 'http://admin.academy.htb:PORT/admin/admin.php?FUZZ=key' -fs <baseline>
```

### 6.B — POST parameters (body, not URL)

```bash
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u http://admin.academy.htb:PORT/admin/admin.php \
  -X POST -d 'FUZZ=key' -H 'Content-Type: application/x-www-form-urlencoded' -fs <baseline>
```

- **PHP only auto-parses `application/x-www-form-urlencoded`.** Drop that header and `$_POST` is empty → every request looks like a miss.
- JSON frameworks (Express): try `-d '{"FUZZ":"key"}' -H 'Content-Type: application/json'`.
- Always try **both GET and POST** before concluding "no fuzzable input" — POST-only params are deliberately hidden = interesting. Re-baseline per method (`POST baseline ≠ GET baseline`).
- A response that changes from "no access" to `Invalid id!` confirms the param is wired.

**Output checkpoint:** a confirmed parameter name (e.g. `user` / `id` / `username`) → fuzz its value, Phase 7.

Detail: `[[10-parameter-fuzzing-get]]`, `[[11-parameter-fuzzing-post]]`.

---

## Phase 7 — Value Fuzzing → Flag

**Goal:** find the magic value that triggers the privileged code path. Closing step: directory → page → param → **value** → flag.

| Need | Build the wordlist |
|---|---|
| Numeric IDs | `seq 1 1000 > ids.txt` |
| Zero-padded | `for i in $(seq -w 1 1000); do echo $i; done > ids.txt` |
| Year range | `seq 1970 2030 > years.txt` |
| Usernames | `/opt/useful/seclists/Usernames/Names/names.txt` |
| user:pass combos | `for u in $(cat users); do for p in $(cat pw); do echo "$u:$p"; done; done > combos.txt` |

```bash
ffuf -w ids.txt:FUZZ -u http://admin.academy.htb:PORT/admin/admin.php \
  -X POST -d 'id=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -fs <baseline>

# ffuf shows hit/miss only — ALWAYS extract with curl:
curl -s 'http://admin.academy.htb:PORT/admin/admin.php' -X POST -d 'id=73' | grep 'HTB'
```

- A hit with size **slightly** off baseline (768 → 787) is the right divergence. A huge jump = probably a server error page, not success.
- Try lowercase first when fuzzing names; check app-layer case-sensitivity.

**Output checkpoint:** the flag / privileged data extracted via curl+grep. Chain complete.

Detail: `[[12-value-fuzzing]]`, full chain `[[13-skills-assessment]]`.

---

## Decision Tree (Under Exam Pressure)

```
Web port open, need content
│
├─ No directories known ──────────► Phase 1 (dir fuzz, -ic)
│     └─ STUCK >5 min, 0 hits ────► escalate wordlist; try Phase 4 (other vhosts exist?)
│
├─ Have a dir, want files ────────► Phase 2 stage1 (indexFUZZ → ext)
│     └─ ext confirmed ───────────► Phase 2 stage2 (FUZZ.<ext>)
│     └─ STUCK >5 min ────────────► Phase 3 recursion -e .php,.phps,.php7
│
├─ Multi-level / slow manual ─────► Phase 3 (-recursion -recursion-depth 1 -v)
│     └─ runs forever ────────────► cap depth=1; kill+restart on queued subpath
│
├─ Need other sites ──────────────► Phase 4
│     ├─ public domain ───────────► 4.A sub-domain
│     │    └─ Errors:~all / 0 hits ► switch to 4.B
│     └─ lab / *.htb ─────────────► 4.B vhost (/etc/hosts first)
│          └─ every result 200 same size ─► Phase 5 (NOT a dead end — calibrate)
│
├─ Wall of identical 200s ────────► Phase 5: read Size → -fs <n>  (or -ac)
│     └─ baseline keeps shifting ─► page is dynamic → -fl or -fw instead of -fs
│     └─ -ac looks wrong ─────────► manual -fs, compare two runs
│
├─ Have endpoint, no input found ─► Phase 6.A GET (?FUZZ=key) -fs
│     └─ STUCK >5 min, nothing ───► Phase 6.B POST (-X POST -d, +Content-Type!)
│          └─ still nothing ──────► try JSON Content-Type; try Param casing (User/user)
│
└─ Have param name, no result ────► Phase 7 value fuzz (seq/names list) -fs
      └─ found hit ───────────────► curl -s ... | grep HTB   (NEVER trust ffuf body)
      └─ STUCK >5 min ────────────► widen value list (seq 1 10000 / -w bigger), re-baseline
```

---

## Signal → Counter-Move Reference

| Signal / Symptom | Likely Cause | Exact Fix |
|---|---|---|
| `Errors: 4997` / all errors on sub-domain fuzz | No public DNS for lab domain | Switch to vhost fuzz 4.B (`-H 'Host: FUZZ.academy.htb'`) |
| Every result `200 OK`, identical `Size:` | Server returns default-vhost / baseline page | Read baseline, `-fs <n>` or `-ac` (Phase 5) |
| Wordlist's first hits are copyright junk | `directory-list-2.3` comment block fuzzed | Add `-ic` |
| `index..php` candidates | Double dot — `web-extensions.txt` already has the dot | Use `indexFUZZ`, not `index.FUZZ` |
| `200 Size: 0` on a found dir/ext | Page exists but empty / language parsed | Not broken — drill in (Phase 2) / ext is enabled |
| POST fuzz returns all misses on PHP | Missing `Content-Type` → `$_POST` empty | Add `-H 'Content-Type: application/x-www-form-urlencoded'` |
| GET param fuzz finds nothing | Param is POST-only (hidden by design) | Repeat as POST (6.B) |
| `-ac` results inconsistent between runs | Dynamic baseline drift | Manual `-fs`; or `-fl`/`-fw` (line/word stable) |
| Browser can't open vhost ffuf found | No `/etc/hosts` entry | `sudo sh -c 'echo "IP vhost.academy.htb" >> /etc/hosts'` (incl. port when visiting) |
| Recursion never ends | No depth cap / app 200s everything | `-recursion-depth 1`; add filter; kill+restart on subpath |
| Value hit size = huge jump vs baseline | Server error page, not success | Treat slight divergence as the real hit; ignore big jumps |
| Param hits but obvious value missed | App case-sensitivity (`User` vs `user`) | Try both casings; lowercase first for names |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# ── DIRECTORY ──
ffuf -ic -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://$IP:$PORT/FUZZ

# ── EXTENSION (stage 1) then FILENAME (stage 2) ──
ffuf -w /opt/useful/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ -u http://$IP:$PORT/blog/indexFUZZ
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://$IP:$PORT/blog/FUZZ.php

# ── RECURSION + extensions ──
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://$IP:$PORT/FUZZ -recursion -recursion-depth 1 -e .php -v

# ── SUB-DOMAIN (public DNS) ──
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u https://FUZZ.$DOMAIN/

# ── VHOST (no DNS) — /etc/hosts first ──
sudo sh -c 'echo "$IP  academy.htb" >> /etc/hosts'
ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://academy.htb:$PORT/ -H 'Host: FUZZ.academy.htb' -fs $BASELINE

# ── FILTER / CALIBRATE ──
ffuf -w $WORDLIST:FUZZ -u $URL -ac                         # auto
ffuf -w $WORDLIST:FUZZ -u $URL -fs $BASELINE               # by size
ffuf -w $WORDLIST:FUZZ -u $URL -mr "You don't have access!" # match known string

# ── PARAMETER GET / POST ──
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u "http://$VHOST:$PORT/admin/admin.php?FUZZ=key" -fs $BASELINE
ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://$VHOST:$PORT/admin/admin.php -X POST -d 'FUZZ=key' -H 'Content-Type: application/x-www-form-urlencoded' -fs $BASELINE

# ── VALUE FUZZ → extract ──
seq 1 1000 > ids.txt
ffuf -w ids.txt:FUZZ -u http://$VHOST:$PORT/admin/admin.php -X POST -d 'id=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -fs $BASELINE
curl -s http://$VHOST:$PORT/admin/admin.php -X POST -d 'id=73' | grep 'HTB'

# ── FULL CHAIN (skills assessment pattern) ──
ffuf -w .../subdomains-top1million-5000.txt:FUZZ -u http://$IP:$PORT -H 'Host: FUZZ.academy.htb' -fs 985
ffuf -w .../web-extensions.txt:FUZZ -u http://faculty.academy.htb:$PORT/indexFUZZ
ffuf -w .../directory-list-2.3-small.txt:FUZZ -u http://faculty.academy.htb:$PORT/courses/FUZZ -e .php,.phps,.php7 -fs 287 -mr "You don't have access!" -t 100
ffuf -w .../burp-parameter-names.txt:FUZZ -u http://faculty.academy.htb:$PORT/courses/linux-security.php7 -X POST -d 'FUZZ=key' -H 'Content-Type: application/x-www-form-urlencoded' -fs 774 -t 100
ffuf -w .../Usernames/Names/names.txt:FUZZ -u http://faculty.academy.htb:$PORT/courses/linux-security.php7 -X POST -d 'username=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -fs 781 -t 100
curl -s http://faculty.academy.htb:$PORT/courses/linux-security.php7 -X POST -d 'username=harry' | grep "HTB{.*}"
```

---

## Quick Reference — Tools / Flags by Function

| Function | Flag / Tool |
|---|---|
| Bind keyword | `-w <list>:FUZZ` (keyword can be anything: `FUZZ`, `FUZ2Z`) |
| Place keyword | URL path, `?p=FUZZ`, `-d 'FUZZ=v'`, `-H 'Host: FUZZ.x'` |
| Method | `-X POST` (default GET) |
| Body | `-d 'FUZZ=key'` |
| Header | `-H 'Header: val'` |
| Threads | `-t 40` default; `-t 100` lab-only |
| Silent / verbose | `-s` (pipeable) / `-v` (full URLs — needed w/ recursion) |
| Skip comment lines | `-ic` |
| Recursion | `-recursion -recursion-depth N` |
| Extensions | `-e .php,.phps,.php7` |
| Match | `-mc -ms -mw -ml -mr` |
| Filter | `-fc -fs -fw -fl -fr` |
| Auto-calibrate | `-ac` |
| Wordlists (Pwnbox) | dirs `directory-list-2.3-small.txt` · ext `web-extensions.txt` · params `burp-parameter-names.txt` · subs `subdomains-top1million-5000.txt` · names `Usernames/Names/names.txt` (all under `/opt/useful/seclists/Discovery/...`) |
| Locate a wordlist | `locate directory-list-2.3-small.txt` |
| Extract result | `curl -s <url> [-X POST -d ...] | grep HTB` |

---

## Top Gotchas

1. **ffuf shows hit/miss, never the body.** Always re-fire the winning request with `curl -s … | grep` to get the flag/data.
2. **Baseline changes every stage.** vhost ≠ extension ≠ GET param ≠ POST param ≠ value. Re-read `Size:` and re-set `-fs` each phase.
3. **PHP POST needs `Content-Type: application/x-www-form-urlencoded`.** Forget it and `$_POST` is empty — 100% misses, looks like the param doesn't exist.
4. **`web-extensions.txt` already has the leading dot.** `indexFUZZ` not `index.FUZZ`.
5. **`-ic` or you fuzz the copyright block** of `directory-list-2.3-small.txt`.
6. **`Errors: ~4997` on sub-domain fuzz = no public DNS** → it's not broken, switch to vhost fuzzing.
7. **`200 Size: 0` is not nothing** — page exists / extension parsed. Drill in, don't skip.
8. **Recursion with no `-recursion-depth` cap** can run for hours; depth 2 × multi-ext = exponential. Start at depth 1.
9. **`/etc/hosts` goes stale** after a target restart (IP changes). Edit/delete the old line — don't append a duplicate. It has no concept of ports; add the port when browsing.
10. **GET-only fuzzing misses POST-only params** (and vice versa). Always try both methods before declaring "no input".
11. **`-ac` drifts on dynamic pages** — it re-calibrates per run; compare against a manual `-fs`, or filter on `-fl`/`-fw` (more stable than size).
12. **Massive size jump on a value hit ≠ success** — likely a server error page. The real hit is usually a *slight* divergence from baseline.

---

## Related Vault Notes

- Theory: `[[01-introduction]]`, `[[02-web-fuzzing]]`, `[[06-dns-records]]`, `[[08-vhost-fuzzing]]`, `[[11-parameter-fuzzing-post]]`
- Labs: `[[03-directory-fuzzing]]`, `[[04-page-fuzzing]]`, `[[05-recursive-fuzzing]]`, `[[07-sub-domain-fuzzing]]`, `[[09-filtering-results]]`, `[[10-parameter-fuzzing-get]]`, `[[12-value-fuzzing]]`, `[[13-skills-assessment]]`
- Chains into: `[[../web-recon/00-METHODOLOGY]]` (vhost/subdomain hand-off), `[[../web-attacks/00-METHODOLOGY]]` & `[[../attacking-common-applications/00-METHODOLOGY]]` (once content/params found), `[[../file-inclusion/00-METHODOLOGY]]` (discovered `?file=`-style params)

Triage by symptom: [[../ATTACK-PATHS]]
