## ID
600

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 1 — Introduction to Windows Privilege Escalation

## Description
Explains why privilege escalation is a critical post-exploitation step, covering common goals, real-world scenarios, attack surfaces, and the importance of manual enumeration skills.

## Tags
theory, windows, privilege-escalation, methodology, introduction, concept

## TL;DR — What's Important
- **Goal of Windows privilege escalation:** Reach Local Administrator or NT AUTHORITY\SYSTEM, or sometimes another specific user, to gain persistence, access local resources, or move laterally.
- **Why it happens:** Personnel shortages, lack of patching, weak configurations, and insufficient internal auditing leave paths open.
- **Vast attack surface:** Exploits range from group/user privilege abuse, UAC bypass, weak service/file permissions, kernel exploits, credential theft, to traffic capture.
- **Manual skills are essential:** In locked-down environments (no internet, blocked USB) you must rely on manual PowerShell and command-line enumeration, not just automated tools.
- **Real-world adaptability:** Scenarios often require creative chaining—like mounting .VHDX files from open shares to extract credentials, or abusing SeImpersonatePrivilege via database links.

## Concept Overview
Windows privilege escalation is the process of increasing your access level on a compromised host after initial foothold. It’s a core component of nearly every penetration test, enabling persistence, lateral movement, and access to sensitive data. Common end goals are membership in the local Administrators group or running code as NT AUTHORITY\SYSTEM. The prevalence of privilege escalation flaws stems from resource constraints, weak configurations, and missed patches, turning it into a consistently necessary and creative phase of an engagement.

## Key Concepts

### Reasons for Privilege Escalation
1. **Gold image testing** – Validate client workstation/server builds for misconfigurations.
2. **Local resource access** – Gain access to databases or sensitive files that the current user cannot read.
3. **Domain foothold** – Elevate to SYSTEM on a domain-joined machine to pivot into Active Directory.
4. **Credential harvesting** – Obtain higher-privilege credentials for lateral movement or further escalation.

### Common Attack Surface Examples
- Abusing Windows group privileges (e.g., Backup Operators, Server Operators)
- Abusing Windows user privileges (e.g., SeImpersonate, SeAssignPrimaryToken)
- Bypassing User Account Control (UAC)
- Weak service or file permissions
- Unpatched kernel exploits
- Credential theft (LSASS dumping, SAM/SECURITY hive extraction)
- Network traffic capture (e.g., LLMNR/NBT-NS poisoning)

## Why It Matters
In real engagements, privilege escalation is often the only way to reach critical systems or demonstrate impact beyond a low-privilege shell. The scenarios in this module show that success frequently depends on chaining simple misconfigurations (open share → VHD mount → credential extraction) or leveraging a single privileged token. Mastering both manual and tool-based techniques ensures you can operate regardless of network restrictions, and understanding the “why” behind each attack helps you adapt when the textbook path is blocked.

## Defender Perspective
- **Detection:** Monitor for suspicious process creation (e.g., cmd.exe spawning from a service), token manipulation (SeImpersonate abuse), LSASS access, and unexpected local account creation. Look for known privilege escalation tool signatures.
- **Common mitigations:** Apply least privilege, enforce UAC, enable Credential Guard, perform regular internal assessments, audit file share permissions, and promptly patch OS and software.
- **MITRE ATT&CK:** Tactic TA0004 (Privilege Escalation) – techniques such as Exploitation for Privilege Escalation (T1068), Valid Accounts (T1078), Access Token Manipulation (T1134), and Credential Dumping (T1003).

## Key Takeaways
- Privilege escalation is not a linear checklist; it demands deep understanding of Windows internals, token behavior, and file system quirks.
- Manual enumeration (icacls, whoami /priv, schtasks, PowerShell) is your lifeline in restricted environments where tools can’t be loaded.
- A single misconfiguration often opens a chain of opportunities—always look for ways to combine findings (share + VHD + offline registry parsing).
- The line between privilege escalation and credential theft is thin; extracting a local admin hash from a gold image can compromise an entire fleet.
- Even when code execution is limited, primitive operations like mounting a drive or connecting to a database can lead to full compromise.

## Gotchas
- Adding a new local user is extremely noisy and should be a last resort; prefer token manipulation, service abuse, or password extraction.
- Kernel exploits can crash unstable systems—test them in a lab first when possible.
- UAC bypass techniques vary wildly across Windows versions; always confirm the OS build before relying on a specific method.
- LSASS memory dumps may trigger antivirus or EDR; consider using built-in tools like `comsvcs.dll` or living-off-the-land techniques to minimize footprint.