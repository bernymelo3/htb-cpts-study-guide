## ID
104

## Module
Attacking Common Services

## Kind
notes

## Title
Section 4 — Finding Sensitive Information

## Description
Why every piece of data – a filename, an email address, a configuration entry – can be the critical clue that leads from low‑privileged access to remote code execution, using a realistic attack chain across FTP, email, and databases.

## Tags
theory, enumeration, data-leakage, pivoting, credential-hunting

## TL;DR — What's Important
- **Treat every piece of information as a potential credential:** Usernames, email addresses, DNS records, source code snippets, config files – all can be reused elsewhere.
- **One misconfigured service (e.g., anonymous FTP) can reveal a seemingly trivial piece of data** (a filename `johnsmith`) that unlocks a completely different service (email login with the same username).
- **Email is a treasure trove:** Search for “password”, “creds”, “credentials”, “attachment”, “confidential” – internal emails often contain plaintext passwords or password reset links.
- **Databases hold the crown jewels:** Once inside, enumerate database names, table names, and columns. Look for tables named `users`, `credentials`, `passwords`, `sysadmin`.
- **Understand the target’s business to know what sensitive information looks like:** A healthcare client → PHI; a fintech → credit card numbers; a software company → source code and API keys.

## Concept Overview
Finding sensitive information is not just about running `grep -r password`. It requires a mindset: every byte you can read from a service is a potential stepping stone. The attack chain in this section’s introduction shows how a single empty file named `johnsmith` on an anonymous FTP server led to email access (using `johnsmith` as a username), then to an email containing MSSQL credentials, and finally to RCE on the database server. Sensitive information can be explicit (passwords in config files) or implicit (usernames, server names, internal IP ranges, software versions). The skill is connecting seemingly unrelated data points across different services to escalate privileges or pivot to new hosts.

## Key Concepts

### Categories of Sensitive Information
| Category | Examples | Where to Find |
|----------|----------|----------------|
| Identifiers | Usernames, email addresses, employee IDs | File shares, email signatures, database `user` tables |
| Authentication secrets | Passwords, NTLM hashes, API keys, SSH keys | Config files, bash history, registry, LSASS dumps |
| Infrastructure data | DNS records, internal IP ranges, hostnames | Email headers, log files, SMB share names |
| Business data | PII, financial records, source code, customer lists | Databases, file shares, SharePoint |
| Configuration leakage | Connection strings, service accounts, registry keys | Web.config, appsettings.json, .env files, `/etc/` |

### The Attack Chain Mindset
1. **Initial access** – Any access, no matter how limited (e.g., anonymous FTP read‑only).
2. **Harvest all data** – List everything. Pay special attention to files that look like they contain names, emails, or configuration details.
3. **Reuse discovered strings as credentials** – Try every username/email you find against every other service (SMB, email, database, RDP, SSH).
4. **Search inside files** – Once you have access to a file share or email mailbox, search for keywords (`password`, `cred`, `key`, `secret`, `token`).
5. **Use found credentials to access higher‑value services** – Database access often leads to command execution; email access leads to password reset links.
6. **Rinse and repeat** – New service → new data → new credentials → new service.

### Why Small Details Matter
- A filename `johnsmith` suggests a username pattern (first initial + last name, or full name). Try `jsmith`, `john.smith`, `johnsmith` against every login form.
- An email signature reveals employee names, titles, phone numbers – use them for password spraying or social engineering.
- A configuration file with an IP address and port number (`db.internal:3306`) tells you where to pivot.
- A log file showing a failed login attempt with a username `svc_backup` – that username is likely valid somewhere.

## Why It Matters
In penetration testing, the difference between a failed assessment and a successful one is often not technical skill but **correlation**. Many testers enumerate a service, note the results, and move on without asking “What can I do with this?” The example chain is realistic: an anonymous FTP share with a seemingly useless file named after a person. Most testers would ignore it. But trying that name as a username against the email server worked, and the email server contained the real prize. Sensitive information is rarely labeled `passwords.txt`; it is hidden in plain sight. Learning to spot and connect these clues turns a limited foothold into domain compromise.

## Defender Perspective
- **Detection:** Data Loss Prevention (DLP) systems can alert on sensitive strings (`password=`, `BEGIN RSA PRIVATE KEY`) being written to world‑readable shares. Monitor for unusual file access patterns (e.g., a user reading thousands of files in a short time).
- **Mitigation:**  
  - Enforce strict access controls – no anonymous authentication anywhere.  
  - Use file classification labels (e.g., Microsoft Information Protection) to mark sensitive documents, and restrict reading them to only necessary personnel.  
  - Regularly scan file shares and email archives for exposed credentials using tools like `Credential Digger` or `Whispers`.  
  - Implement email filtering to block outgoing messages containing “password” + “attached” unless explicitly approved.  
  - For databases, always encrypt columns containing PII or credentials at rest.
- **MITRE ATT&CK:** T1083 (File and Directory Discovery – hunting through shares), T1113 (Screen Capture – may capture sensitive info from RDP sessions), T1552 (Unsecured Credentials: Files, Registry, Bash History).

## Key Takeaways
- Build a “sensitive data” cheat sheet: strings to search for in any text file (`password`, `pwd`, `secret`, `key`, `token`, `api_key`, `connectionString`, `SA password`, `service_account`).
- When you find a username, try it against every service on the network. Password reuse is rampant.
- Email is often overlooked during internal assessments. If you compromise a user’s workstation, read their Outlook `.ost` file or use `mailparser` on exported PSTs. Internal emails are a goldmine of plaintext credentials and network diagrams.
- In databases, after basic enumeration (`SELECT name FROM master..sysdatabases`), always query for tables with common names: `users`, `credentials`, `passwords`, `sysadmins`, `employee`, `customer`. Then `SELECT * FROM <table>`.
- The most sensitive information is often not data at rest but data in transit – capture network traffic (tcpdump, Wireshark) to spot cleartext protocols (FTP, HTTP, Telnet, SMTP without STARTTLS).

## Gotchas
- Don’t only search for English keywords. If the target is in a non‑English country, also search for local translations (`senha`, `passwort`, `motdepasse`, `contraseña`).
- Encrypted files (`.kdbx`, `.gpg`, `.enc`) are not useless – they indicate that there is a password protected file somewhere. If you can find a key file or a memory dump containing the password, you may decrypt it.
- Searching inside databases can be slow. Use `WHERE` clauses to limit to text columns or use `LIKE '%password%'` only on columns of type `varchar`, `nvarchar`, `text`.
- Beware of honeypots: a file named `passwords.txt` that contains fake credentials might be a trap. Verify by trying the credentials on a low‑impact service first.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|common-services]]  
← [[03-service-misconfigurations]] | [[06-ftp-latest-vulns]] →
<!-- AUTO-LINKS-END -->
