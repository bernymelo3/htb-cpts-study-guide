## ID
603

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 13 — Hyper-V Administrators

## Description
Explains why Hyper-V Administrators can easily become Domain Admins on virtualized domain controllers by extracting credentials from offline VHDX disks, and how to abuse the group membership to gain SYSTEM on any Hyper-V host via legacy hard link abuse.

## Tags
theory, windows, hyper-v, privilege-escalation, group-membership, credential-theft

## TL;DR — What's Important
- **Virtualized DCs = Domain Admins for Hyper-V Admins:** With full Hyper-V access, an admin can clone or mount the domain controller's virtual disk offline and extract the NTDS.dit file, yielding all domain credentials.
- **Hard link vulnerability (pre-March 2020):** The `vmms.exe` process restores file permissions after VM deletion without impersonation, allowing an attacker to create a hard link to a chosen SYSTEM-owned file and gain full control of it.
- **Practical path to SYSTEM:** By hard-linking a protected executable like `C:\Program Files (x86)\Mozilla Maintenance Service\maintenanceservice.exe`, you can replace it with a malicious binary and start the associated service as `NT AUTHORITY\SYSTEM`.
- **Mitigated by March 2020 security updates:** The hard link behavior was changed, so the exploitation vector no longer works on fully patched systems; always check OS build and patch level before attempting.
- **Group membership is extremely powerful:** Even without hard link tricks, the ability to manipulate virtual disks makes this group equivalent to Domain Admins if Domain Controllers are virtualized.

## Concept Overview
The Hyper-V Administrators group grants full control over Hyper-V hosts and all virtual machines (VMs). If an organization virtualizes its Domain Controllers, any member of this group can clone or mount the Domain Controller’s virtual hard disk offline and extract the entire Active Directory database (NTDS.dit). This instantly escalates the user to Domain Admin level. Additionally, on systems vulnerable to CVE-2018-0952 or CVE-2019-0841, Hyper-V Administrators can exploit a time-of-check to time-of-use (TOCTOU) hard link weakness in `vmms.exe` to gain `NT AUTHORITY\SYSTEM` privileges on the Hyper-V host itself, even if no Domain Controller is present.

## Key Concepts

### Attack Path 1: Offline VHD Mount – Stealing the Domain
- Hyper-V Administrators can shut down, export, or clone the VM that runs a Domain Controller.
- The virtual hard disk file (`.vhdx`) can be mounted on any system where the attacker has administrative access (e.g., their own workstation or a compromised host).
- Once mounted, the attacker can read the `NTDS.dit` file and the SYSTEM registry hive, then use tools like `secretsdump.py` to extract all domain user hashes (including Domain Admins and krbtgt).
- This attack requires no code execution on the DC itself and leaves minimal forensic evidence inside the running VM.

### Attack Path 2: Hard Link Abusing vmms.exe (Pre-March 2020)
When a VM is deleted, `vmms.exe` (the Hyper-V Virtual Machine Management Service) attempts to restore the original file permissions on the corresponding `.vhdx` file. It performs this operation as `NT AUTHORITY\SYSTEM` **without impersonating the user** who requested the deletion.

This creates a privilege escalation window:

1. **Delete** a VM’s `.vhdx` file (the attacker, as a Hyper-V Admin, has permission to do so).
2. **Create a native hard link** that points the now-“deleted” `.vhdx` path to a protected SYSTEM-owned file you want to control (e.g., the binary of a service that runs as SYSTEM and can be started by unprivileged users).
3. When `vmms.exe` tries to restore permissions on the `.vhdx` file, it actually changes the permissions on the target SYSTEM file to give the attacker Full Control.
4. The attacker then takes ownership, replaces the target file with a malicious executable, and starts/restarts the corresponding service to execute code as `NT AUTHORITY\SYSTEM`.

### Practical Example: Mozilla Maintenance Service
A common target for this technique is:
