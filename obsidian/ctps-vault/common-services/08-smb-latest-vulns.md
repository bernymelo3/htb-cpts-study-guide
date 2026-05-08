## ID
108

## Module
Attacking Common Services

## Kind
notes

## Title
Section 8 — Latest SMB Vulnerabilities

## Description
CVE-2020-0796 (SMBGhost) – an integer overflow vulnerability in SMBv3.1.1 compression that allows unauthenticated remote code execution on Windows 10 versions 1903/1909.

## Tags
theory, vulnerability, smb, rce, integer-overflow, cve

## TL;DR — What's Important
- **CVE-2020-0796** affects SMBv3.1.1 compression mechanism on Windows 10 1903/1909.
- **Attack vector:** Unauthenticated attacker sends a malformed compressed message during SMB session negotiation.
- **Root cause:** Integer overflow due to missing bounds checks – the size of compressed data exceeds variable limits, overwriting memory and CPU instructions.
- **Impact:** Remote code execution (RCE) with SYSTEM privileges – full compromise of the target.
- **Concept pattern:** Source = crafted request → Process = compression handling → Privileges = SYSTEM → Destination = local memory (first cycle), then second cycle leads to attacker‑controlled code execution.

## Concept Overview
SMBGhost (CVE-2020-0796) is a critical vulnerability in the SMBv3.1.1 protocol implementation on Windows 10 versions 1903 and 1909. The flaw resides in the compression mechanism used during SMB session negotiation. An unauthenticated attacker can send a specially crafted compressed message that triggers an integer overflow. This overflow allows overwriting kernel memory and redirecting execution flow to attacker‑supplied code, leading to remote code execution with SYSTEM privileges. The vulnerability is “wormable” – it can spread from one vulnerable machine to another without user interaction.

## Key Concepts

### Integer Overflow
- Occurs when an arithmetic operation produces a value too large to fit in the allocated memory space (e.g., a 32‑bit integer exceeds 2³²‑1).
- In SMBGhost, the lack of bounds checking on compressed data sizes leads to an integer overflow, causing the system to misinterpret the length of data to copy.

### Buffer Overflow / Memory Corruption
- After the integer overflow, the system writes data into a buffer that is too small, overwriting adjacent memory.
- Overwritten memory includes function pointers or CPU instructions – the attacker can replace them with malicious shellcode.

### Compression Mechanism in SMBv3.1.1
- SMBv3.1.1 introduced optional compression to improve performance.
- The client and server negotiate compression capabilities during the session setup.
- The vulnerability is triggered by sending a malformed compressed message before authentication.

### Attack Cycle (as described in the module)

| Step | SMBGhost Phase | Concept Category |
|------|----------------|------------------|
| 1 | Attacker sends manipulated compressed request | Source |
| 2 | SMB server processes the packet according to negotiated protocol | Process |
| 3 | Processing occurs with SYSTEM / administrator privileges | Privileges |
| 4 | Local process handling compressed packets becomes the initial Destination | Destination (local) |

**Second cycle – RCE:**

| Step | RCE Phase | Concept Category |
|------|-----------|------------------|
| 5 | The overwritten buffer becomes the Source for new instructions | Source |
| 6 | Integer overflow occurs, forcing CPU to execute attacker’s instructions | Process |
| 7 | Same SYSTEM privileges are used | Privileges |
| 8 | Attacker gains remote access to the system (reverse shell, etc.) | Destination (network) |

## Why It Matters
- SMBGhost is a **wormable** vulnerability – similar to EternalBlue (MS17-010) that fueled WannaCry.
- No authentication required – any unauthenticated attacker on the network can compromise vulnerable systems.
- The vulnerability exists in a widely used protocol (SMB) on millions of Windows machines (especially legacy or unpatched environments).
- For penetration testers, SMBGhost is a high‑impact finding that leads directly to full system compromise.

## Defender Perspective
- **Detection:** Monitor SMB traffic for anomalous compression negotiation packets. Look for unexpected SMBv3.1.1 compression attempts from unauthenticated sources. EDR signatures exist for known SMBGhost exploit attempts.
- **Mitigation:**  
  - Apply Microsoft security patch KB4551762 (March 2020) or later.  
  - Disable SMBv3.1.1 compression via PowerShell:  
    `Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters" -Name "DisableCompression" -Type DWORD -Value 1 -Force`  
  - Block SMB traffic (TCP port 445) at firewalls where not needed.
- **MITRE ATT&CK:** T1210 (Exploitation of Remote Services), T1068 (Exploitation for Privilege Escalation – though here RCE is direct).

## Key Takeaways
- Complex vulnerabilities (integer overflow, buffer overflow) still follow the same **Source → Process → Privileges → Destination** pattern.
- Even without understanding low‑level exploitation, you can recognise that SMB compression handling is a **Process** that lacked bounds checking (Source validation).
- Privileges matter: SMB service runs as SYSTEM – any RCE yields full control.
- The Destination in the first cycle is **local memory** (overwritten buffer). The second cycle’s Destination is the **attacker’s remote system** (reverse shell).
- Always check for missing patches on internal assessments – SMBGhost is a quick win on unpatched Windows 10 1903/1909.

## Gotchas
- SMBGhost only affects **SMBv3.1.1** with compression enabled. SMBv1/v2 are not vulnerable.
- The vulnerability was patched in March 2020. Many enterprises may still have unpatched hosts, especially in air‑gapped or slow‑to‑update environments.
- Public exploits exist (e.g., Python, PowerShell, Metasploit module `exploit/windows/smb/smbghost`). However, exploitation can cause blue screens (BSOD) – be careful in production assessments.
- The vulnerability requires the target to have SMB listening on port 445. Many organisations block SMB inbound at the perimeter, but internally it is often open.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|common-services]]  
← [[07-attacking-smb]] | [[11-attacking-rdp]] →
<!-- AUTO-LINKS-END -->
