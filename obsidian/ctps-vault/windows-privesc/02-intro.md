## ID
601

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 2 — Useful Tools

## Description
Introduces a curated set of Windows privilege escalation enumeration tools, their purposes, and the critical balance between automation and manual technique mastery for both penetration testers and defenders.

## Tags
theory, windows, privilege-escalation, tools, enumeration, methodology

## TL;DR — What's Important
- **Tools accelerate enumeration:** Seatbelt, winPEAS, PowerUp, SharpUp, JAWS, and others quickly surface misconfigurations, missing patches, and credential stores.
- **Manual skills are non-negotiable:** In air-gapped or restricted environments, you must rely on built-in commands and manual analysis; tools are not always an option.
- **Tools are a double-edged sword:** They produce immense output, can overwhelm with irrelevant data, and may generate false positives/negatives; interpretation skill is key.
- **Detection is almost certain:** Well-known tools like LaZagne are flagged by 47/70+ AV engines; bypass techniques exist but assume you'll get caught on a monitored engagement.
- **Defensive value:** The same tools help sysadmins and internal teams find low-hanging fruit before an assessment, hardening gold images and validating patch levels.

## Concept Overview
Privilege escalation enumeration can be tedious and complex, given the vast Windows attack surface. To manage this, a range of free, open-source tools have been developed to automate checks for common misconfigurations, missing patches, and credential storage. This section provides an overview of the most essential tools, from C# scanners like Seatbelt and SharpUp to PowerShell scripts like PowerUp and JAWS, and specialized tools like SessionGopher and LaZagne. It emphasizes that while tools save time, they must be paired with deep manual understanding to handle false results, constrained environments, and evasive tradecraft.

## Key Concepts

### Core Toolset Overview
| Tool | Description |
|------|-------------|
| **Seatbelt** | C# multi-check scanner covering local privilege escalation vectors |
| **winPEAS** | Exhaustive enumeration script; color-coded output flags potential privilege paths |
| **PowerUp** | PowerShell script that detects and can exploit common misconfigurations |
| **SharpUp** | C# equivalent of PowerUp |
| **JAWS** | PowerShell 2.0-compatible enumeration script, useful on older systems |
| **SessionGopher** | Extracts and decrypts saved sessions from PuTTY, WinSCP, FileZilla, RDP, etc. |
| **Watson** | .NET tool that maps missing KBs to known privilege escalation exploits |
| **LaZagne** | Comprehensive local password extraction from browsers, databases, email, Wi-Fi, etc. |
| **WES-NG** | Windows Exploit Suggester Next Generation; takes `systeminfo` output to recommend exploits |
| **Sysinternals Suite** | Includes AccessChk, PipeList, PsService for granular permission and service enumeration |

### Tool vs. Manual Enumeration Tradeoffs
- **Automation:** Fast, comprehensive, good for initial triage – but can be noisy, blocked by AV, and produce overwhelming output.
- **Manual methods:** Stealthier, works in any environment, builds deep understanding – but slower and more prone to human oversight.
- **Ideal approach:** Use tools for broad scanning, manual verification for interesting leads, and always be able to fall back to manual techniques.

### Common Risks and Limitations
- **Detection:** Most tools are signatured by AV/EDR; running them in production without prior client agreement may end the engagement early.
- **Information overload:** winPEAS output can span hundreds of lines; learning to filter on red/yellow findings is critical.
- **False positives/negatives:** Tools may flag a misconfiguration that isn't exploitable, or miss a nuanced chain; manual checks are needed to confirm.
- **System impact:** Aggressive enumeration (WMI queries, volume shadow copy enumeration) can strain fragile systems; use with care.

## Why It Matters
In time-pressured assessments, relying solely on manual enumeration leads to missed opportunities, while blindly trusting automated output can result in dead ends or false confidence. Knowing what each tool does, how to compile/obfuscate it, and when *not* to use it is a hallmark of a seasoned tester. For defenders and administrators, these same tools offer a free way to self-assess, hardening environments against the exact attacks that will be thrown at them.

## Defender Perspective
- **Detection:** Antivirus and EDR will flag most precompiled versions of these tools. Behavioral monitoring can detect suspicious enumeration activity (e.g., rapid WMI queries, SAM hive access) even if the binary itself is obfuscated.
- **Mitigations:** Perform internal assessments proactively using these tools, harden gold images, restrict user write access (e.g., don't allow BUILTIN\Users to write in `C:\Windows\Temp`), and monitor for unexpected use of Sysinternals tools.
- **MITRE ATT&CK:** Primarily maps to TA0004 (Privilege Escalation) and TA0007 (Discovery) – uses tools like Seatbelt and winPEAS for system information discovery (T1082), permission enumeration (T1069), and credential access (T1003).

## Key Takeaways
- The tool list is a starting point; learning to read their source code reveals the exact Windows APIs and registry keys they query, deepening your understanding of privilege escalation mechanics.
- Even on an engagement where AV is disabled, using tools like LaZagne or PowerUp can trigger advanced EDR behavioral detections—assume everything is watched unless confirmed otherwise.
- The same tool can be an attacker's scanner and a defender's audit weapon; internal teams who run winPEAS regularly will find misconfigurations before a pentester does.
- In environments where you can't upload binaries, you can still perform many checks via PowerShell or even VBScript/CMD; JAWS was literally designed for that.
- Always have fallback plans: if WES-NG is blocked, you can manually cross-check installed patches against a known vulnerability list.

## Gotchas
- Precompiled binaries from unofficial sources can be flagged or even backdoored; always compile from source if using on a client system.
- Writing temporary tools to `C:\Windows\Temp` is standard, but some hardening removes BUILTIN\Users write access—check with `icacls` first.
- LaZagne may trigger an immediate incident response; never use it without explicit client permission, even for testing.
- Watson and WES-NG rely on accurate patch enumeration; if `systeminfo` output is truncated or filtered, they may miss critical vulns.