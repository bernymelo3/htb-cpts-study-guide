## ID
101

## Module
Attacking Common Services

## Kind
notes

## Title
Section 1 — Interacting with Common Services

## Description
How to connect to SMB, email, and database services using native Windows/Linux CLIs and GUIs – the essential enumeration and interaction skills before any attack.

## Tags
theory, methodology, enumeration, smb, databases, email

## TL;DR — What's Important
- **Know the default tools:** `dir` / `net use` on Windows, `mount -t cifs` on Linux for SMB; `sqsh` / `sqlcmd` for MSSQL; `mysql` for MySQL.
- **Mounting vs. native interaction:** Mounting an SMB share lets you use native tools (`find`, `grep`, `Get-ChildItem`) – far more powerful than one‑off commands.
- **Credential objects in PowerShell:** Use `ConvertTo-SecureString` + `New-Object PSCredential` – you cannot pass plaintext passwords directly to `New-PSDrive`.
- **Protocol matters for email:** IMAP vs. POP3 vs. SMTP – each uses different ports and encryption (TLS vs STARTTLS). Always check supported authentication types.
- **CLI speed vs. GUI depth:** CLIs automate bulk searches (e.g., counting 29k files in seconds). GUIs (dbeaver, SSMS) help with schema exploration and ad‑hoc queries.

## Concept Overview
Interacting with a service means establishing a client‑server connection using the correct protocol, authentication, and tooling. This section covers the three most common enterprise service categories – file sharing (SMB), email (IMAP/POP3/SMTP), and relational databases (MSSQL/MySQL) – from both Windows and Linux attack hosts. The goal is to become fluent in the native commands and popular third‑party tools so enumeration and exploitation are not slowed down by basic connectivity issues.

## Key Concepts

### Comparison Table – SMB Interaction
| Platform | Native CLI | Mounting | Credential Handling |
|----------|-----------|----------|---------------------|
| Windows (CMD) | `net use`, `dir` | Map drive letter | `/user:` flag |
| Windows (PS) | `New-PSDrive` | Mount as PS drive | `PSCredential` object |
| Linux | `mount -t cifs` | Mount to `/mnt` | credentials file or `-o username=,password=` |

### Definitions
- **CIFS** – Common Internet File System, the protocol dialect used by modern SMB implementations. Linux `mount -t cifs` speaks SMB 2.0/3.0.
- **PSCredential** – PowerShell’s secure object for passing usernames and encrypted passwords to cmdlets.
- **IMAPS / SMTPS** – Email protocols wrapped in TLS from the first packet (dedicated ports 993, 465).  
- **STARTTLS** – Upgrades a plaintext connection to TLS after the initial handshake (ports 143, 587).

### Categories / Types of Database Interaction
- **Command‑line utilities** – `mysql`, `sqsh`, `sqlcmd`. Fast for scripting and one‑off queries.
- **GUI applications** – dbeaver, SSMS, MySQL Workbench. Better for schema browsing, multi‑table joins, and visual query building.
- **Programming language drivers** – `pyodbc`, `pymysql`. Used in custom exploits or automation scripts.

## Why It Matters
Before you can exploit a misconfiguration or crack a hash, you must be able to connect to the service. Many penetration testers waste time fumbling with syntax or missing the right tool for the job. Knowing, for example, that `sqsh -S` expects a host:port and that `mysql -h` requires the port flag (`-P`) separately, or that Linux needs `cifs-utils` installed before mounting – these small details determine whether you enumerate successfully or stare at “connection refused”. Additionally, once you mount a file share, you can apply your entire grep/find/Get-ChildItem expertise to hunt for credentials across thousands of files in seconds.

## Defender Perspective
- **Detection:** Unusual mounting of SMB shares from non‑domain IPs, or repeated authentication failures across multiple protocols, triggers alerts in modern EDR/SIEM (e.g., Zeek SMB logs, Windows Event ID 4625).
- **Mitigation:** Disable SMBv1, enforce SMB signing, use network segmentation to restrict access to file shares and databases. For email, disable plaintext authentication and require STARTTLS.
- **MITRE ATT&CK:** T1046 (Network Service Scanning), T1047 (Windows Management Instrumentation – often used with databases), T1110 (Brute Force – credential stuffing via CLI tools).

## Key Takeaways
- The same interaction pattern applies across services: identify protocol → choose native or third‑party client → authenticate → browse/query. Once you master one (e.g., SMB), you can transfer the logic to FTP, NFS, or even cloud storage.
- `dbeaver` is a force multiplier – one GUI for MSSQL, MySQL, PostgreSQL, and many others. Installing it on your attack VM saves time hopping between different database CLIs.
- Error messages are gold. “NT_STATUS_ACCESS_DENIED” vs. “NT_STATUS_BAD_NETWORK_NAME” tell you whether the share exists but you lack permissions, or it doesn’t exist at all.
- For email, always try `telnet <server> 25` and `EHLO` – the server’s response reveals supported authentication methods and STARTTLS availability without any client software.

## Gotchas
- On Linux, `mount -t cifs` without `nounix, noserverino` may fail against Windows servers. Add `-o vers=3.0` or `vers=2.0` if version negotiation fails.
- PowerShell’s `New-PSDrive` with `-Credential` still mounts the drive, but subsequent `Get-ChildItem` may ignore the credential and use your current logon session. Use `-Persist` or access via `\\server\share` directly.
- Evolution email client on Linux may fail with “bwrap: Can't create file” – set `export WEBKIT_FORCE_SANDBOX=0` before launching.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|common-services]]  
[[02-attack-concept]] →
<!-- AUTO-LINKS-END -->
