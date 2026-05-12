# NOTE — MSSQL

## ID
116

## Module
Footprinting

## Kind
notes

## Title
Section 14 — MSSQL

## Description
Microsoft SQL Server (TCP/1433) enumeration: NMAP ms-sql-* scripts, Impacket mssqlclient.py with `-windows-auth`, Metasploit's `mssql_ping`, and listing non-default databases via `sys.databases`.

## Tags
mssql, microsoft-sql-server, port-1433, ssms, impacket, mssqlclient, metasploit, windows-auth, t-sql

## Commands
- `locate mssqlclient` — find Impacket's mssqlclient.py
- `python3 mssqlclient.py <user>@<IP> -windows-auth` — connect via Windows auth
- `/usr/bin/impacket-mssqlclient <user>@<IP> -windows-auth` — Pwnbox path
- `sudo nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-xp-cmdshell,ms-sql-config,ms-sql-ntlm-info,ms-sql-tables,ms-sql-hasdbaccess,ms-sql-dac,ms-sql-dump-hashes --script-args mssql.instance-port=1433,mssql.username=sa,mssql.password=,mssql.instance-name=MSSQLSERVER -sV -p 1433 <IP>`
- Metasploit: `use auxiliary/scanner/mssql/mssql_ping; set rhosts <IP>; run`
- Inside the SQL session: `select name from sys.databases`

## Concept Overview
MSSQL = Microsoft's closed-source RDBMS. Ships with Windows Server, but also runs on Linux/macOS. Strong native .NET integration → very common backend for legacy in-house apps. Default port **TCP/1433**. **DAC** (Dedicated Admin Connection) listens on **TCP/1434** — separate channel for emergency admin access.

### Ways to talk to it
- **SSMS** (SQL Server Management Studio) — Windows GUI client, often **installed on the DBA's workstation**, not just the server. Can leak saved creds.
- **mssql-cli** — cross-platform CLI
- **SQL Server PowerShell** — `Invoke-Sqlcmd`
- **HeidiSQL / SQLPro** — third-party GUIs
- **Impacket `mssqlclient.py`** — pentest go-to (CLI, supports `-windows-auth`, NTLM, hashes)

### Default System Databases
| Database | Purpose |
|----------|---------|
| `master` | Tracks all system info for an instance — server-wide config |
| `model` | Template for new DBs — settings here propagate to every new DB |
| `msdb` | SQL Server Agent jobs + alerts |
| `tempdb` | Temporary objects |
| `resource` | Read-only system objects |

A non-default DB (anything not in this list) = the actual application data.

## Default Configuration
- Service runs as `NT SERVICE\MSSQLSERVER`.
- Default auth = **Windows Authentication** — login goes through the local SAM or AD DC.
- Encryption is NOT enforced by default — clients can connect plaintext.
- AD-integrated auth = great for audit/control but means a compromised AD account = SQL access too.

## Dangerous Settings / Vectors
- **Clients connecting unencrypted** — sniffable creds + queries.
- **Self-signed certs** when encryption *is* on — spoofable, MitM possible.
- **Named pipes enabled** — alternative attack surface (`\\<host>\pipe\sql\query`).
- **`sa` enabled with weak/default password** — admins forget to disable it after install.
- **`xp_cmdshell` enabled** — RCE primitive directly from a query.
- **`xp_dirtree`** — SMB UNC path coercion → NetNTLMv2 hash capture (Responder).

## Methodology
1. **NMAP scripted scan** — gives hostname, instance name, version, named pipe, NTLM info (computer/domain name even without auth).
2. **Metasploit `mssql_ping`** — quick + clean version/instance/port discovery.
3. **Try guessed creds** with mssqlclient.py — `Administrator`, `sa`, app-name accounts, `backdoor`, etc. Always test `-windows-auth` (NTLM) AND SQL auth.
4. **Once authenticated**: `select name from sys.databases` — separate user DBs from system DBs. Pivot into the user DB.
5. **From there**: enumerate tables (`SELECT name FROM sys.tables`), describe columns, exfil interesting data.
6. **Privilege escalation** — `EXEC xp_cmdshell 'whoami'` if enabled, or use `xp_dirtree` to coerce auth back to a Responder listener.

## Important NSE Scripts
| Script | What it does |
|--------|-------------|
| `ms-sql-info` | Hostname, instance, version, named pipe |
| `ms-sql-ntlm-info` | Computer + domain name via NTLM challenge — works pre-auth |
| `ms-sql-empty-password` | Test empty password for `sa` and others |
| `ms-sql-config` | Server config dump (auth required) |
| `ms-sql-tables` | Enumerate tables (auth required) |
| `ms-sql-hasdbaccess` | Per-user DB access (auth required) |
| `ms-sql-dac` | Probe DAC port 1434 |
| `ms-sql-dump-hashes` | Extract password hashes (auth required, sysadmin) |
| `ms-sql-xp-cmdshell` | Test/run xp_cmdshell (auth required) |

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — MSSQL server hostname | **`ILF-SQL-01`** | `sudo nmap --script ms-sql-info,ms-sql-ntlm-info,... -sV -p 1433 <IP>` → `Windows server name: ILF-SQL-01` (also visible in `ms-sql-ntlm-info`) |
| Q2 — Non-default DB on the server | **`Employees`** | `/usr/bin/impacket-mssqlclient backdoor@<IP> -windows-auth` (creds `backdoor:Password1`) → `SELECT name from sys.databases` → returns master/tempdb/model/msdb/**Employees** |

### Reference command
```bash
/usr/bin/impacket-mssqlclient backdoor@<IP> -windows-auth
SQL> SELECT name from sys.databases
```

## Key Takeaways
- `ms-sql-ntlm-info` leaks computer + DNS domain **pre-auth** via the NTLM challenge — free recon.
- Impacket's `mssqlclient.py -windows-auth` is the workhorse — supports NTLM hashes via `-hashes`, Kerberos via `-k`.
- Skip the four system DBs to find the real target — anything else in `sys.databases` is application data.
- SSMS often installed on **dev/admin workstations**, not the server. Saved-connection files there can leak creds.

## Gotchas
- Encryption negotiation can fail silently against legacy installs — `-windows-auth` over plaintext may not work; try without.
- `mssql_ping` is UDP/1434 (SQL Browser) — if SQL Browser is disabled, you must already know the instance + port.
- DAC connections require a sysadmin login AND the DAC remote-access option enabled — usually disabled by default.
- xp_cmdshell is disabled by default in modern installs; enable via `sp_configure 'xp_cmdshell', 1; RECONFIGURE;` (sysadmin only — leaves audit trail).

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[13-mysql]] | [[15-oracle-tns]] →
<!-- AUTO-LINKS-END -->
