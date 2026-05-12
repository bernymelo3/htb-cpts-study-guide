# NOTE — MySQL

## ID
112

## Module
Footprinting

## Kind
notes

## Title
Section 13 — MySQL

## Description
MySQL/MariaDB enumeration on TCP/3306: nmap mysql-* NSE scripts (with caution — false positives are common), client login, schema/table walk, and MySQL-specific dangerous settings.

## Tags
mysql, mariadb, mssql, port-3306, nmap, sql, lamp, lemp

## Commands
- `mysql -u <user> -p<password> -h <IP>` — note: no space between `-p` and password
- `sudo nmap <IP> -sV -sC -p3306 --script mysql*` — enumerate (verify by hand!)
- Inside mysql:
  - `show databases;`
  - `use <db>;`
  - `show tables;`
  - `show columns from <table>;`
  - `select * from <table>;`
  - `select * from <table> where <col> = "<str>";`
  - `select version();`

## Concept Overview
MySQL = Oracle-owned open-source RDBMS, common back-end of LAMP/LEMP stacks (especially WordPress). MariaDB = compatible fork after Oracle's MySQL acquisition. Default port **TCP/3306**. Ideally bound to localhost; often exposed in dev/staging "temporarily forever".

### Stack Notes
- LAMP = Linux + Apache + MySQL + PHP
- LEMP = Linux + Nginx + MySQL + PHP
- WordPress, MediaWiki, Joomla — all default to MySQL backend stored on localhost.

## Default Configuration — `/etc/mysql/mysql.conf.d/mysqld.cnf`
| Field | Default |
|-------|---------|
| `port` | 3306 |
| `bind-address` | `127.0.0.1` (default — change at your peril) |
| `socket` | `/var/run/mysqld/mysqld.sock` |
| `datadir` | `/var/lib/mysql` |
| `user` | `mysql` |

## Dangerous Settings
| Setting | Why dangerous |
|---------|--------------|
| `user` | If misconfigured (e.g. `root`) MySQL runs with too many OS privileges |
| `password` | Plaintext in config — anyone with file read = creds |
| `admin_address` | Admin TCP listener — if exposed externally, separate attack surface |
| `debug` | Verbose error output, often returned to web app users |
| `sql_warnings` | Single-row INSERT warnings expose data |
| `secure_file_priv` | If unset/empty, `LOAD_FILE` and `INTO OUTFILE` work anywhere — file read/write primitive |

`debug` + `sql_warnings` exposed via web errors → SQLi feedback channel for blind injection becomes verbose injection.

## Default System Databases
| Database | Purpose |
|----------|---------|
| `information_schema` | ANSI/ISO standard metadata (tables, columns, privileges) |
| `mysql` | Internal — users, grants, system tables |
| `performance_schema` | Performance metrics |
| `sys` | Convenience views over the above (Microsoft-style system catalog) |

When credentials work, dump `select host, user, authentication_string from mysql.user;` for further pivot/cracking.

## Methodology
1. **Nmap with script suite** — `--script mysql*` runs `mysql-info`, `mysql-empty-password`, `mysql-brute`, `mysql-enum`, `mysql-databases`, `mysql-dump-hashes`. Watch for false positives.
2. **Manual verify** — `mysql -u root -h <IP>` to confirm. Nmap "empty password root" is famously over-eager.
3. **Try guessed passwords** found elsewhere (web app config files, GitHub leaks, default `P4SSw0rd`-tier).
4. **Once authenticated** — `show databases;` → enumerate user-data DBs (skip `information_schema`/`mysql`/`performance_schema`/`sys` until last).
5. **Dump interesting tables** — `select * from users;` etc. Look for plaintext or weak hashes.
6. **Hash extract** — `select host, user, authentication_string from mysql.user;` then crack offline (hashcat).

## Important NSE Scripts
| Script | Purpose |
|--------|--------|
| `mysql-info` | Version, capabilities, salt |
| `mysql-empty-password` | Test empty password for common users |
| `mysql-enum` | User existence enumeration |
| `mysql-brute` | Credential brute |
| `mysql-databases` | Lists DBs (auth required) |
| `mysql-dump-hashes` | Dumps password hashes (auth required) |
| `mysql-users` | Lists users (auth required) |
| `mysql-vuln-cve2012-2122` | Auth-bypass CVE check |

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — MySQL version (`MySQL X.X.XX`) | **`MySQL 8.0.27`** | `sudo nmap -p3306 -sV <IP>` → `MySQL 8.0.27-0ubuntu0.20.04.1` |
| Q2 — Email of customer "Otto Lang" | **`ultrices@google.htb`** | `mysql -u robin -probin -h <IP>` (creds `robin:robin`) → `show databases;` → `use customers;` → `describe myTable;` → `SELECT email FROM myTable WHERE name = "Otto Lang";` |

## Key Takeaways
- **Always verify Nmap's "empty password" finding manually** — this script returns false positives constantly.
- `-p<password>` must have **no space** between flag and value. `mysql -u root -p P4SS` ≠ `mysql -u root -pP4SS`.
- `mysql.user` table is the cred-extraction goldmine when authenticated.
- `secure_file_priv` empty + `FILE` privilege = read/write any file the mysql user can.

## Gotchas
- `bind-address = 0.0.0.0` is rare but devastating — exposes the whole DB to the network.
- Newer MySQL (8.x) requires explicit `--ssl-mode=DISABLED` to talk to legacy clients.
- MariaDB/MySQL prompt looks identical — you may be talking to MariaDB without realising. Check `select version();` to confirm.
- Some Nmap scripts (`mysql-databases`, `mysql-dump-hashes`) only run after a successful brute — they'll silently fail otherwise.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[12-snmp]] | [[14-mssql]] →
<!-- AUTO-LINKS-END -->
