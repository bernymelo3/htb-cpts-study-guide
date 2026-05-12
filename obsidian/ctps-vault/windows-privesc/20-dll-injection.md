## ID
603

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 13 — Hyper-V Administrators

## Description
Explains how Hyper-V Administrators can compromise virtualized Domain Controllers by offline disk mounting and leverage a legacy hard-link weakness in vmms.exe to escalate to SYSTEM on the virtualization host.

## Tags
theory, windows, hyper-v, group-membership, privilege-escalation, credential-theft

## TL;DR — What's Important
- **Hyper-V Admin on a virtualized DC equals Domain Admin:** You can clone or export the DC’s virtual disk, mount it offline, and extract the NTDS.dit file to dump all domain password hashes.
- **Hard-link race in vmms.exe (pre-March 2020):** When you delete a VM’s `.vhdx`, the Hyper‑V service restores file permissions as SYSTEM without impersonation, allowing you to hijack arbitrary SYSTEM-owned files by creating a hard link.
- **Privilege escalation path to SYSTEM:** Replace a hijacked service binary (e.g., `maintenanceservice.exe`) with a payload, start the service, and get code execution as NT AUTHORITY\SYSTEM.
- **Patch boundary is March 2020:** Windows security updates changed hard‑link behaviour, fully neutering the vmms.exe attack. Unpatched systems remain vulnerable to CVE-2018-0952 or CVE-2019-0841.
- **Even without the hard‑link trick, group membership alone is critical:** Full control over VMs allows credential theft that directly compromises the whole domain.

## Concept Overview
The Hyper‑V Administrators group grants complete control over Hyper‑V hosts and their virtual machines. If an organisation virtualises its Domain Controllers, any member of this group effectively holds Domain Admins‑level power because they can shut down, clone, or snapshot a DC, then mount the virtual hard disk offline and extract the `NTDS.dit` database. Additionally, on unpatched systems, a TOCTOU (time‑of‑check to time‑of‑use) vulnerability in the Hyper‑V management service (vmms.exe) allows an attacker to take ownership of any SYSTEM‑owned file by creating a hard link where the service expects a `.vhdx` file. By targeting a service binary that runs as SYSTEM, an attacker can replace it with a malicious executable and achieve full local SYSTEM access.

## Key Concepts

### Offline Credential Theft from Virtualised Domain Controllers
- **Access gained:** Hyper‑V Administrators can export, clone, or snapshot a running/virtualised Domain Controller.
- **Extraction technique:** The VM’s `.vhdx` (or `.vhd`) virtual disk is mounted on any machine where the attacker has administrative privileges.
- **What to steal:** Inside the mounted disk, the `NTDS.dit` Active Directory database and the SYSTEM registry hive are copied.
- **Result:** Running `secretsdump.py` offline against the dumped files reveals all domain user NTLM hashes, including the krbtgt account, instantly granting full domain compromise.
- **Stealth:** No malicious code is executed on the actual Domain Controller, making this method hard to detect with traditional endpoint monitoring.

### Hard‑Link Privilege Escalation on the Hyper‑V Host (CVE-2018-0952 / CVE-2019-0841)
- **Root cause:** When a virtual machine is deleted, `vmms.exe` (running as NT AUTHORITY\SYSTEM) restores the original file permissions on the deleted `.vhdx` file. It performs this operation **without impersonating the user**, so it acts with SYSTEM permissions on whatever file currently sits at that path.
- **Exploit chain:**
  1. **Delete** any VM’s `.vhdx` file (allowed for Hyper‑V Admins).
  2. **Create a native hard link** from the deleted `.vhdx` path to a protected SYSTEM file you want to control (e.g., `C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe`).
  3. When `vmms.exe` “restores” permissions, it inadvertently grants the attacker Full Control over the target SYSTEM file.
  4. **Take ownership** of the target file (`takeown /F <file>`).
  5. **Replace** the file with a malicious executable of your choice.
  6. **Start** the associated service (e.g., `sc start MozillaMaintenance`) to execute the payload as SYSTEM.

### Practical Example: Hijacking Mozilla Maintenance Service
- Firefox installs the `MozillaMaintenance` service, which runs as SYSTEM and is startable by unprivileged users.
- After the hard‑link trick, the attacker has Full Control of `maintenanceservice.exe`.
- Replacing it with a custom binary and starting the service yields a SYSTEM shell.
- The same concept works for any service that meets the criteria: runs as SYSTEM, startable by unprivileged users, and has an executable located on the same volume as the `.vhdx`.

**Important:** The March 2020 Windows security updates changed the file permission restoration behaviour so that `vmms.exe` only operates on the original `.vhdx` file object, breaking the hard‑link attack. Fully patched systems are not vulnerable to this method.

## Why It Matters
During a penetration test, finding your account in the Hyper‑V Administrators group immediately opens a low‑noise path to Domain Admin if Domain Controllers are virtualised. On a standalone Hyper‑V host or when domain pivoting is not an option, the hard‑link escalation technique (if the OS is not up‑to‑date) delivers SYSTEM access without requiring sophisticated kernel exploits or credential harvesting. Understanding both avenues ensures you can maximise the impact of this group membership, whether the target is a single server or an entire domain.

## Defender Perspective
- **Detection:** Monitor group membership changes in Hyper‑V Administrators. Log VM export and deletion events (e.g., via the Microsoft‑Windows‑Hyper‑V‑* event logs). For the hard‑link attack, flag unusual hard links created in directories containing service executables (e.g., `C:\Program Files\`). After March 2020, this specific technique is impossible; still watch for behavioural signs such as unexpected replacement of signed service binaries.
- **Mitigations:**
  - Apply the March 2020 (or later) cumulative updates to close the hard‑link race condition.
  - Avoid virtualising Domain Controllers on the same Hyper‑V cluster that general admins manage. Use separate, highly controlled administration tiers (Tier 0) or keep at least one physical DC.
  - Use Shielded VMs to protect VM disks from being mounted by unauthorised users. Note that local Hyper‑V admins may still be able to break shielding.
  - Restrict Hyper‑V Administrators membership via Just‑In‑Time (JIT) privileged access management and audit all changes.
  - Implement LAPS to prevent lateral movement from a compromised Hyper‑V host, and protect offline backups of Domain Controllers.
- **MITRE ATT&CK:** TA0004 (Privilege Escalation) – Valid Accounts (T1078) via group abuse; Credential Dumping (T1003) via NTDS; Access Token Manipulation (T1134) if SYSTEM is obtained through service replacement; File and Directory Permissions Modification (T1222) for hard‑link trick.

## Key Takeaways
- The Hyper‑V Administrator group is often overlooked but is effectively a Domain Admin factory if DCs are virtualised. Always check for virtualised Domain Controllers during enumeration.
- Offline NTDS extraction does not require any malicious code on the DC itself, making it extremely attractive for stealthy operations.
- The hard‑link exploit requires a service startable by authenticated users – many third‑party services (Mozilla, Adobe, etc.) inadvertently provide this opportunity.
- The technique was patched globally, not per‑feature; all affected Windows versions received the March 2020 fix. Still, it remains a classic example of TOCTOU and hard‑link abuse that repurposes a benign permission restoration into a privilege escalation.
- Even after patching, the group’s ability to re‑create or move virtual disks can be chained with other weaknesses (e.g., insecure disk mounts, backup access) to reach SYSTEM.

## Gotchas
- The hard‑link technique requires the target executable and the `.vhdx` to reside on the **same volume** (usually C:); be mindful of file system location.
- Firefox (and thus the Mozilla Maintenance Service) must be installed on the target for that particular example. Scan for other services meeting the same criteria if Firefox is absent.
- Never work on the **only copy** of a Domain Controller’s VHD if you plan to restore it; use a clone or snapshot to avoid damaging the production environment and causing USN rollback issues.
- The technique is loud if the service replacement triggers an application crash or gets detected by endpoint protection; have a fallback plan for less monitored services.
- Check the OS patch level with `systeminfo` or `wmic qfe` before attempting the hard‑link exploit; on a fully updated system, pivot directly to offline NTDS extraction instead.