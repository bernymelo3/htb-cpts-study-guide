# THEORY — Introduction

## ID
615

## Module
File Transfers

## Kind
theory

## Title
Section 1 — Introduction

## Description
Why file transfers matter mid-engagement and the real-world chain of fallbacks (PowerShell → certutil → FTP → SMB) when host/network defenses block the obvious paths.

## Tags
file-transfer, intro, methodology, opsec, network-defenses

## TL;DR — What's Important
- File transfer is a constant need mid-engagement (upload tools in, exfil loot out).
- Host controls (AV/EDR, AppLocker, WDAC) and network controls (firewall egress rules, IDS/IPS, web filtering) will block most "obvious" methods.
- You always need a **plan B/C/D** — the right method depends on which protocol the firewall lets out (HTTP, FTP, SMB, DNS, ICMP, SSH).

## Setting the Stage (canonical scenario from the module)
1. RCE on IIS via unrestricted upload → drop web shell → reverse shell.
2. Try to fetch `PowerUp.ps1` via PowerShell → blocked by Application Control Policy.
3. Manual enum finds `SeImpersonatePrivilege` → need to transfer `PrintSpoofer`.
4. Try `certutil` from own GitHub → blocked by web content filtering.
5. Try Windows FTP client → outbound TCP/21 blocked at firewall.
6. Set up `impacket-smbserver` on TCP/445 → **allowed** → transfer succeeds → privesc to admin.

## Key Takeaways
- Knowing **multiple** transfer methods is the skill — any single technique can be blocked. Practice the fallbacks.
- Host controls and network controls are independent — the binary may run but the port may be blocked, or the port is open but AppLocker kills the binary.
- This module is a **reference** — bookmark it; you'll come back to it during AD, PrivEsc, Shells & Payloads, and Pivoting labs.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-transfers]]
[[02-windows-file-transfer-methods]] →
<!-- AUTO-LINKS-END -->
