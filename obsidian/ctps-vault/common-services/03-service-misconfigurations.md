## ID
103

## Module
Attacking Common Services

## Kind
notes

## Title
Section 3 — Service Misconfigurations

## Description
The most common security holes in enterprise services: default/weak credentials, anonymous authentication, overly permissive access rights, and unnecessary default features – and how to spot them quickly.

## Tags
theory, misconfiguration, default-credentials, access-control, enumeration

## TL;DR — What's Important
- **Default credentials still work:** `admin:admin`, `root:<blank>`, `administrator:Password` – always try these first. Vendors still ship with them, and admins often forget to change them.
- **Anonymous authentication is a goldmine:** If a service (FTP, SMB, HTTP) allows login with no password, enumerate everything. It often reveals credentials for other services.
- **Access rights misconfigurations give you more than intended:** A user meant only to upload files might also be able to read the entire share, including config files with database passwords.
- **Unnecessary defaults increase attack surface:** Debug interfaces, sample pages, unused ports, and default accounts are often left enabled because “it worked out of the box.”
- **Prevention is repeatable hardening:** Use automation (Ansible, hardening scripts) to ensure every environment (dev, QA, prod) is identically locked down.

## Concept Overview
Service misconfigurations occur when an administrator, developer, or automated installer fails to change insecure default settings or grants excessive permissions. Unlike a software vulnerability (e.g., buffer overflow), misconfigurations are not bugs – they are operational errors. However, they are far more common in real‑world assessments than zero‑days. The OWASP Top 10 lists “Security Misconfiguration” as a separate category because it consistently appears in breaches. Typical examples include leaving default admin credentials enabled, allowing anonymous authentication, granting users more privileges than their role requires, and failing to disable debugging or sample pages after deployment.

## Key Concepts

### Comparison Table – Misconfiguration Types
| Type | Example | Impact | How to Detect |
|------|---------|--------|----------------|
| Default credentials | `ftp:ftp` on FTP server | Full access to the service | Try known default lists (e.g., `seclists` / Default Credentials) |
| Weak/blank passwords | `admin:<blank>` on router admin interface | Unauthorized administrative control | Attempt login with common weak passwords |
| Anonymous authentication | SMB share allows `null session` | Read/write files without any account | `smbclient -N -L //target` or `crackmapexec smb target -u '' -p ''` |
| Overly permissive ACLs | User `joe` can read HR share | Sensitive data exposure | Enumerate shares and test file access with low‑privilege creds |
| Unnecessary features enabled | Debug mode left on in web app | Stack traces reveal file paths, SQL queries | Trigger errors (invalid input) and observe responses |

### How to Systematically Check for Misconfigurations
1. **Banner grab** – Identify service and version. Check if default credentials are known (e.g., `searchsploit` or `https://default-password.info/`).
2. **Test anonymous / null session** – For SMB, FTP, HTTP, LDAP.
3. **Attempt common weak credentials** – Use a small wordlist (`admin`, `password`, `123456`, `<blank>`).
4. **If you get any access, enumerate thoroughly** – List directories, read config files, look for password reuse.
5. **Check for unnecessary enabled features** – Debug endpoints, example pages, writable directories, exposed admin panels.

## Why It Matters
In a typical internal penetration test, misconfigurations are the #1 way testers gain initial footholds. A default password on a Jenkins server leads to script execution. An anonymous FTP upload folder leads to a webshell. An overly permissive SMB share contains a `credentials.txt` file with domain admin credentials. These are not sophisticated attacks – they are the result of lazy deployment. Understanding misconfigurations means you stop looking for complex exploits and start checking the obvious things first. The time saved is enormous.

## Defender Perspective
- **Detection:** Monitor authentication logs for failed logins using default usernames (`admin`, `root`, `guest`). Use vulnerability scanners (Nessus, OpenVAS) with a policy that checks for default credentials. For SMB, look for Event ID 4625 with blank username or well‑known defaults.
- **Mitigation:**  
  - Enforce a password policy that rejects weak and default passwords.  
  - Disable anonymous/guest access on all services unless explicitly required.  
  - Implement a hardening baseline (CIS benchmarks, DISA STIG) and verify it with automated tools (e.g., OpenSCAP, InSpec).  
  - Remove or disable all sample applications, documentation, and debug interfaces before production deployment.  
  - Use configuration management (Ansible, Puppet) to ensure every host is identical and re‑hardened after updates.
- **MITRE ATT&CK:** T1078 (Valid Accounts – default accounts), T1110.001 (Password Guessing), T1133 (External Remote Services – misconfigured VPN/RDP often uses weak creds).

## Key Takeaways
- Default credential lists are not just for CTFs – real enterprises still use them. Build a curated list of the top 20 default username/password pairs and try them against every service you encounter.
- “Anonymous” often means “no password” but can also mean a built‑in guest account. For SMB, test both `''` (empty) and `'guest'`.
- A single misconfiguration can be the key to the entire domain. Never ignore a low‑privilege file share – it might contain an `.rdp` file with a saved password or a `.kdbx` Keepass database.
- When you find a misconfiguration, document the **exact** steps to reproduce. Many will be fixed before your report is delivered, but the pattern (e.g., “all four SharePoint servers had default credentials”) shows systemic failure.
- For defenders, automated scanning once a month is not enough. Configuration drift happens daily. Use continuous compliance tools (e.g., AWS Config, Azure Policy, or OPA) to alert on deviations.

## Gotchas
- Some services treat blank password differently from null authentication. Always try both `-u '' -p ''` and `-u 'guest' -p ''`.
- Default credentials might be per‑vendor, not per‑service. For example, many IoT devices use `root:default` or `admin:1234`. Check `https://cirt.net/passwords` or `https://default-password.info/` for obscure hardware.
- Anonymous authentication on SMB might allow connection but not list shares. Use `smbclient -L //target -N` to force share enumeration.
- Unnecessary defaults include not just features but also default file permissions. A web server might ship with world‑writable log directories – check those too.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|common-services]]  
← [[02-attack-concept]] | [[04-finding-sensitive-info]] →
<!-- AUTO-LINKS-END -->
