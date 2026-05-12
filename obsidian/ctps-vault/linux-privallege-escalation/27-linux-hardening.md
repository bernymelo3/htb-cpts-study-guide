## ID
534

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 27 — Linux Hardening

## Description
Theory section covering defensive hardening measures against Linux privilege escalation, including patching, configuration management, user management, auditing standards, and the Lynis automated audit tool.

## Tags
hardening, defense, lynis, audit, configuration, linux

## Commands
- `./lynis audit system`
- `find / -perm -4000 2>/dev/null`
- `dpkg -l | grep unattended-upgrades`

## What This Section Covers
This section flips perspective from attacker to defender, outlining the hardening measures that would prevent or mitigate every privesc technique covered in the module. It covers four pillars: updates and patching (unattended-upgrades, yum-cron), configuration management (SUID audits, cron/sudo absolute paths, credential hygiene, SELinux), user management (least privilege, PAM password policies, group audits), and periodic auditing using frameworks like DISA STIGs, ISO27001, PCI-DSS and tools like Lynis.

## Methodology
1. **Patching**: Enable automatic updates (`unattended-upgrades` on Debian/Ubuntu, `yum-cron` on RHEL) to eliminate kernel and service-level exploits.
2. **Configuration management**: Audit SUID binaries, ensure cron jobs and sudoers entries use absolute paths, remove cleartext credentials from world-readable files, clean bash history, verify custom library permissions, remove unnecessary packages/services, and consider SELinux.
3. **User management**: Minimize user and admin accounts, enforce strong passphrase policies via PAM, restrict sudo by least privilege, prevent password reuse with `/etc/security/opasswd`, and audit group memberships.
4. **Automation**: Use configuration management tools (Puppet, SaltStack, Zabbix, Nagios) to automate checks, push alerts, and auto-remediate issues across fleets. Use checksum verification for sensitive binaries.
5. **Auditing**: Follow security baselines (DISA STIGs, ISO27001, PCI-DSS, HIPAA) as reference guides. Run Lynis (`./lynis audit system`) for automated configuration audits — it produces a hardening index, warnings, and actionable suggestions. Run as root for more thorough checks.
6. **Remember**: Audits and configuration reviews supplement but do not replace penetration testing.

## Key Takeaways
- Most privesc paths in this module (SUID abuse, cron misconfig, writable scripts, cleartext creds, kernel exploits) are preventable with basic hardening — patching + least privilege + SUID audits eliminates the majority.
- Lynis is a practical tool for both defenders (baseline checks) and pentesters (quick enumeration of misconfigurations from the defender's perspective) — a hardening index of 60/100 like in the example leaves significant attack surface.
- Automatic updates (`unattended-upgrades`) have been default on Ubuntu since 18.04 — if they're disabled or absent, that's a red flag during assessment.
- Configuration management automation (Puppet, Zabbix, Nagios) can detect and remediate drift at scale, but only if the baselines are properly defined — a weak baseline automated across a fleet just scales the weakness.
- Compliance frameworks (ISO27001, PCI-DSS) should inform but not define a security program — passing a controls audit with the bare minimum is not the same as being secure.
- As a pentester, understanding what defenders should do helps you identify what they haven't done — every hardening step skipped is a potential privesc vector.
