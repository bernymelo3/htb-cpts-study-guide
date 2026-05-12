## ID
605

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 31 — Windows Hardening

## Description
Comprehensive coverage of Windows hardening strategies—secure imaging, patch management, Group Policy, user management, auditing, logging, and key configuration measures—to eliminate or drastically reduce local privilege escalation vectors.

## Tags
theory, windows, hardening, defense, best-practices, configuration-management

## TL;DR — What's Important
- **Start with a clean, custom OS image:** Remove bloatware, pre-configure security settings, integrate tested updates, and ensure every deployed host starts from a known-good baseline.
- **Systematic patch management is non-negotiable:** Use WSUS or Group Policy to control Windows Update Orchestrator; test updates in a dev environment before broad deployment; regular reboots finalize patches.
- **Group Policy enforces configuration at scale:** Centrally manage security settings, password policies, Windows Defender behavior, and user preferences across an entire domain or locally.
- **Limit accounts and enforce strong authentication:** Enforce password history, complexity, rotation, and 2FA; audit group memberships to remove excessive privileges; log all login attempts.
- **Deploy Sysmon and centralize logging:** Sysmon enriches Event Logs with process creation, network connections, and file operations; ship logs to a SIEM for correlation and threat hunting.
- **Apply practical hardening measures incrementally:** Secure Boot, BitLocker, absolute paths in scheduled tasks, credential cleanup, Device/Credential Guard, and periodic audits against STIGs or Microsoft baselines.

## Concept Overview
Windows hardening is the proactive, layered process of reducing the attack surface of a Windows host to prevent or severely hinder privilege escalation. It spans the entire system lifecycle—from building a secure, minimal installation image, through continuous patch management and configuration enforcement via Group Policy, to strict user/credential management, comprehensive audit programs, and detailed logging with tools like Sysmon. No single measure eliminates all risks; rather, a defense-in-depth approach that combines these controls makes opportunistic and targeted privilege escalation increasingly difficult. Hardening is equally critical for small businesses and enterprises because attackers no longer discriminate based on organization size.

## Key Concepts

### Secure Clean OS Installation
- Build a custom image from a verified clean ISO (obtained from Microsoft or Media Creation Tool).
- Strip vendor-installed bloatware and preinstalled trial software.
- Pre-load all approved applications required for daily duties.
- Integrate major and minor updates that have been tested in your environment.
- Deploy via Windows Deployment Services (WDS) or System Center Configuration Manager (SCCM) for consistency across the fleet.
- **Result:** Every host starts from an identical, secure baseline, simplifying troubleshooting, updates, and auditing.

### Updates and Patch Management
The Windows Update Orchestrator follows a staged process:
1. Scans the host and checks against Microsoft Update servers or an internal WSUS server.
2. Randomized polling intervals prevent server flooding.
3. Applicable updates are identified and downloaded to a temp folder.
4. Manifests are validated; only necessary files are pulled.
5. Orchestrator calls the installer agent with an action list.
6. Updates are applied but not finalized until a reboot.
7. A reboot finalizes modifications to services and critical settings.

- **Enterprise practices:** Use Windows Server Update Services (WSUS) to centralize downloads and control rollout. Test all updates on a small dev/ring group before fleet-wide deployment. Never push blindly—an update can break a legacy LOB application.
- **Group Policy control:** Configure automatic update behavior, active hours, and restart notifications via Administrative Templates.

### Configuration Management via Group Policy
- **Active Directory-based Group Policy:** Central management of user and computer settings across the domain using the Group Policy Management Console (GPMC) or PowerShell.
- **Local Group Policy:** For standalone machines, `gpedit.msc` provides local control.
- **Scope of control:** Password policies, account lockout, Windows Defender scan schedule, firewall rules, desktop restrictions, browser settings, software installation, and literally thousands of other settings.
- **Granularity:** Plan carefully; poorly tested GPOs can break functionality across the entire domain.

### User Management and Authentication
- **Minimize accounts:** Limit the number of user and administrator accounts per system. Monitor and log all valid/invalid login attempts.
- **Password Policy** (Configured at `Computer Configuration\Windows Settings\Security Settings\Account Policies\Password Policy`):
  - Enforce password history (remember last 6–24 passwords).
  - Minimum password age and complexity requirements.
  - Regular rotation (with caveats: overly frequent changes can lead to weaker real-world passwords).
- **Two-Factor Authentication (2FA):** Combines something you know (password) with something you have (token, authenticator app). Significantly reduces credential reuse and phishing impact.
- **Excessive privilege audit:** Regularly check that users are not in groups like Domain Admins, Backup Operators, or Server Operators unless absolutely needed. Enforce login restrictions on administrative accounts (deny RDP/WinRM, restrict to specific machines).

### Auditing and Compliance Frameworks
- **Security baselines:** DISA STIGs (Security Technical Implementation Guides) provide granular, prescriptive checklists for hardening each Windows version. The STIG Viewer lets auditors step through each rule (Vul ID) and verify compliance.
- **Microsoft Security Compliance Toolkit:** Provides Group Policy baselines aligned with Microsoft's own recommendations.
- **Compliance frameworks (ISO 27001, PCI-DSS, HIPAA):** Help establish security requirements but should supplement, not replace, a tailored security program based on the organization's data, threat model, and operating environment.
- **Audit ≠ Penetration Test:** Configuration audits and STIG checklists are valuable complements, but are often "box-checking exercises" that miss actual exploitable logic flaws. Combine with hands-on testing and vulnerability scanning.

### Logging and Monitoring
- **Sysmon (System Monitor):** Part of Sysinternals Suite. Persistent across reboots. Captures:
  - Process creation with command line hashing.
  - Network connections (source/destination IP, port, process).
  - File creation and deletion with timestamps.
  - Driver and DLL load events.
  - WMI activity.
  - All written to `Applications and Service Logs\Microsoft\Windows\Sysmon\Operational`.
  - Ship to a SIEM for correlation and threat hunting. Use community configs (like SwiftOnSecurity) as a starting point.
- **Network monitoring:** Tools like PacketBeat and IDS/IPS sensors (e.g., Security Onion) collect network traffic for holistic visibility. Combine host and network logs for a complete picture.
- **Event 4672:** Special privileges assigned to new logon—critical for detecting privilege escalation attempts. Tune alerts for sensitive privileges like SeDebug, SeImpersonate, SeTcb.

### Key Hardening Measures Checklist
(Not exhaustive, but covers high-impact, low-effort items)

1. **Enable Secure Boot and BitLocker disk encryption.**
2. **Audit writable files/directories**, especially those in PATH and service binary locations.
3. **Use absolute paths** in all scheduled tasks and scripts that run with elevated privileges.
4. **Eliminate cleartext credentials** from world-readable files, shared drives, and scripts.
5. **Clean up home directories** and clear PowerShell history (`ConsoleHost_history.txt`).
6. **Restrict low-privileged users** from modifying custom DLLs or libraries called by elevated programs.
7. **Remove unnecessary packages and services** (IIS, SMBv1, Print Spooler if not needed) to reduce attack surface.
8. **Enable Device Guard and Credential Guard** on Windows 10/Server 2016+ where hardware virtualization supports it.
9. **Use Group Policy** to enforce all configuration changes consistently.

## Why It Matters
Hardening is the direct countermeasure to every privilege escalation technique covered in the module. A single missed Step—like a writable service binary path or a cleartext credential in a backup script—undoes years of perimeter investment. For penetration testers, understanding the defender's playbook is essential: you must be able to explain not only how you escalated, but exactly which hardening measures would have prevented it in a cost-effective, operationally feasible way. This transforms a penetration test from a "gotcha" list into a actionable remediation roadmap.

## Defender Perspective
- **This entire section is the defender perspective.** It outlines the proactive controls that prevent local privilege escalation.
- **Implementation guidance:** Do not blindly apply every STIG or baseline—assess business impact. Implement measures in waves, testing in dev/staging environments. Combine automated configuration scanning (Nessus, Nessus compliance checks) with hands-on validation.
- **Continual improvement:** Train staff on new attack trends and PoCs. Regularly revisit hardening baselines as operating systems evolve.
- **MITRE ATT&CK mapping for defense:**
  - M1018 (User Account Management) – enforce least privilege.
  - M1027 (Password Policies) – strong passwords, 2FA.
  - M1026 (Privileged Account Management) – restrict admin logins, eliminate excessive group membership.
  - M1051 (Update Software) – systematic patching.
  - M1047 (Audit) – regular configuration reviews against STIGs.
  - M1041 (Encrypt Sensitive Information) – BitLocker, EFS.
  - M1040 (Behavior Prevention on Endpoint) – Credential Guard, Device Guard.

## Key Takeaways
- The most effective hardening measure is a clean, custom image deployed across all hosts—it eliminates random pre-installed vulnerabilities and ensures a known-good baseline.
- Patch management is a process, not a product; WSUS and Group Policy are tools, but testing, deployment rings, and reboot discipline are the real success factors.
- Group Policy is both a hardening tool and a potential vulnerability vector; ensure GPOs themselves are protected and GPO Creator Owner group is tightly controlled.
- Sysmon is free, lightweight, and transforms Windows event logging from sparse to forensic-grade—deploy it everywhere you can.
- Security baselines (STIGs, Microsoft SCT) are a starting point; tailor them to the organization's actual risk profile and test for operational conflicts.
- The line between defender and attacker knowledge is thin: mastering how to harden teaches you exactly which misconfigurations to hunt for during a penetration test.

## Gotchas
- Applying a STIG wholesale without testing can break critical applications—always test in a representative environment first.
- Enforcing frequent password changes without complexity/length requirements often leads to patterns like "Summer2026!"—which are easily cracked.
- Group Policy updates are not instantaneous; changes may take up to 90 minutes (plus background refresh interval) to propagate unless `gpupdate /force` is run.
- 2FA is not a silver bullet; if RDP/NLA is not configured properly, credentials can still be relayed (NTLM relay) even with 2FA for interactive logons.
- Sysmon requires a well-tuned configuration to avoid overwhelming log volume; the SwiftOnSecurity config is a solid foundation, but fine-tune based on your environment's noise level.