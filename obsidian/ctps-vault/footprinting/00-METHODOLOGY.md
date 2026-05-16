# METHODOLOGY — Footprinting (Service Enumeration Playbook)

## ID
718

## Module
Footprinting

## Kind
methodology

## Title
Footprinting — Infrastructure & Service Enumeration Methodology

## Description
Exam-day retrieval tool for the Footprinting module: the 6-layer coverage framework plus a port-triggered, per-service enumeration playbook (FTP, SMB, NFS, DNS, SMTP, IMAP/POP3, SNMP, MySQL, MSSQL, Oracle TNS, IPMI, SSH/Rsync/R-services) with exact commands, misconfig checks, and the credential-reuse loop that solves every lab.

## Tags
methodology, footprinting, enumeration, service-enum, banner, port-open, anonymous-login, null-session, zone-transfer, axfr, smb-shares, nfs-mount, snmp-walk, vrfy, default-creds, credential-reuse, password-reuse, panic, stuck, what-do-i-run

---

## TL;DR — 5-Phase Flow

1. **Passive infra recon** — domain → subdomains (crt.sh), cloud buckets, staff/tech-stack OSINT. Zero packets to target.
2. **Port scan & triage** — `nmap -sV -sC -p-` + targeted UDP. Every open port maps to a service block in Phase 3.
3. **Per-service enumeration** — for each open port, run its block: banner → anonymous/null/default-cred → config/misconfig → loot.
4. **Credential-reuse loop** — every cred/key/hash found is tested against *every other* service. This is the connective tissue of all 3 labs.
5. **Convert to access** — anonymous file read → SSH key → shell; leaked DB cred → query the flag.

---

## Golden Rule + OPSEC Fork

**Golden Rule — find the gap, don't smash the wall.** The 6-layer model (below) is a *coverage framework*, not a checklist. Each layer is a wall with a gap; find the gap that leads to the *next* wall. Don't bash through a hardened service when an anonymous share next to it is wide open. Without the framework you will skip layers (typically Processes/Privileges/OS-Setup) and miss easy wins.

**The 6 Layers (mental model, from `01`/`02`):**

| # | Layer | Goal | Level |
|---|-------|------|-------|
| 1 | Internet Presence | external infra (domains, ASN, cloud) | infrastructure |
| 2 | Gateway | defensive measures (FW, IDS, WAF) | infrastructure |
| 3 | **Accessible Services** | **map & enumerate exposed services** ← *this module lives here* | host |
| 4 | Processes | internal data flows | OS |
| 5 | Privileges | what you can do as which user | OS |
| 6 | OS Setup | internal OS config, sensitive files | OS |

Layers 1–2 are external-only; internal/AD pentests start at a different model. Layers 4–6 are covered by the privesc modules.

**OPSEC Fork — don't waste exam time on:**
- `dig any` (RFC 8482 — filtered on modern resolvers; query record types individually instead).
- Cloud buckets / GitHub leaks unless explicitly in scope (`*.amazonaws.com`, AWS/GCP/Azure host IPs = NOT in scope unless contracted).
- Smashing a hardened service when an anonymous one (FTP/SMB/NFS/Rsync) is open right beside it.
- Trusting Nmap's `mysql-empty-password` / brute scripts — they false-positive constantly; **verify by hand**.
- Treating layers as a strict checklist (brittle) or confusing methodology with toolset (new tool ≠ new methodology).

---

## Phase 1 — Passive Infrastructure Recon

**Goal:** subdomains, IPs, tech stack, leaked secrets — without touching the target.

**Trigger/Precondition:**

| Precondition | Action |
|---|---|
| You have a domain name | crt.sh + dig |
| Found `*.amazonaws.com` / `blob.core.windows.net` / `googleapis.com` host | cloud-bucket dorks (scope permitting) |
| Have a company name | LinkedIn/GitHub staff OSINT for tech stack |

**Commands:**
```bash
# Subdomains from Certificate Transparency
curl -s "https://crt.sh/?q=<DOMAIN>&output=json" | jq . \
  | grep name | cut -d":" -f2 | grep -v "CN=" | cut -d'"' -f2 \
  | awk '{gsub(/\\n/,"\n");}1;' | sort -u

# Resolve each subdomain, keep only company-hosted ones
for i in $(cat subdomainlist); do host $i | grep "has address" | grep <DOMAIN> | cut -d" " -f1,4; done

# DNS records (query types individually — NOT `dig any`)
dig <DOMAIN> A; dig <DOMAIN> MX; dig <DOMAIN> TXT; dig <DOMAIN> NS; dig soa <DOMAIN>

# Cloud-storage dorks (only if in scope)
# Google: intext:<COMPANY> inurl:amazonaws.com   (also blob.core.windows.net / googleapis.com)
```

**Output checkpoint — after this you have:** subdomain→IP list split into company-hosted vs third-party, TXT records revealing SaaS in use, SPF `ip4:` in-scope candidates, admin email from SOA, possible leaked secrets/keys.

---

## Phase 2 — Port Scan & Service Triage

**Goal:** every open port → a Phase 3 service block.

```bash
sudo nmap -sV -sC -p- <IP> --open                 # full TCP, version + default scripts
sudo nmap -sU -sV -p U:53,111,161,623 <IP>        # targeted UDP (DNS/portmapper/SNMP/IPMI)
```

**Port → Phase 3 block:**

| Port(s) | Service | Block |
|---|---|---|
| TCP/21 (or 2121+) | FTP | 3.1 |
| TCP/22 | SSH | 3.12 |
| TCP/25, 587, 465 | SMTP | 3.5 |
| TCP/53, UDP/53 | DNS | 3.4 |
| TCP/110,143,993,995 | IMAP/POP3 | 3.6 |
| TCP/111, 2049, UDP/111 | NFS | 3.3 |
| TCP/139, 445 | SMB | 3.2 |
| TCP/512,513,514 | R-services | 3.12 |
| TCP/873 | Rsync | 3.12 |
| TCP/1433 | MSSQL | 3.8 |
| TCP/1521 | Oracle TNS | 3.9 |
| TCP/3306 | MySQL | 3.7 |
| UDP/161 | SNMP | 3.10 |
| UDP/623 | IPMI | 3.11 |

**Output checkpoint:** a worklist of (port → block) pairs. Work them in parallel; the win is usually one anonymous/default-cred service.

---

## Phase 3 — Per-Service Enumeration

Each block: **banner → anonymous/null/default-cred → config & misconfig → loot.**

### 3.1 — FTP (TCP/21, often non-standard e.g. 2121)
```bash
sudo nmap -sV -p21 -sC -A <IP>                       # add --script-trace to see NSE traffic
nc -nv <IP> 21                                       # banner grab
ftp <IP>                                             # try anonymous:anonymous
wget -m --no-passive ftp://anonymous:anonymous@<IP>  # mirror whole share
openssl s_client -connect <IP>:21 -starttls ftp      # TLS-wrapped FTP + cert
# inside ftp: status / debug / trace / ls -R / get <f> / put <f>
```
- **Misconfig:** `anonymous_enable=YES`, `anon_upload_enable=YES`+web-root = webshell, `ls_recurse_enable=YES`. Config: `/etc/vsftpd.conf`, `/etc/ftpusers`.
- **Gotcha:** vsFTPd 2.3.4 = backdoor CVE; `hide_ids=YES` masks all owners as `ftp`; TFTP can't list — must know filename.
- **Loot:** full tree, downloadable files, version+CVE, cert CN/email.

### 3.2 — SMB (TCP/139, 445)
```bash
smbclient -N -L //<IP>                               # null-session share list
smbclient //<IP>/<share> -N                          # connect (Enter = empty pw)
smbmap -H <IP>                                        # share permissions
crackmapexec smb <IP> --shares -u '' -p ''
enum4linux-ng -A <IP>                                 # automated full recon
rpcclient -U "" <IP>                                  # null RPC
# rpcclient: srvinfo / querydominfo / netshareenumall / netsharegetinfo <s> / enumdomusers / queryuser <RID>
# RID brute when enumdomusers denied:
for i in $(seq 500 1100); do rpcclient -N -U "" <IP> -c "queryuser 0x$(printf '%x\n' $i)" | grep "User Name\|user_rid\|group_rid" && echo ""; done
```
- **Misconfig:** `guest ok = yes`, `read only = no`, `logon script=`, `magic script=`. Config: `/etc/samba/smb.conf`.
- **Gotcha:** `enumdomusers` denied but `queryuser <RID>` allowed → RID-brute. "Login successful" + `ls` error = authed to IPC$ only.
- **Loot:** shares, files, domain name (`querydominfo`), share path/remark (`netsharegetinfo`), users+RIDs, Samba version.

### 3.3 — NFS (TCP/111, 2049)
```bash
sudo nmap --script nfs* <IP> -sV -p111,2049
showmount -e <IP>                                    # list exports
mkdir target-NFS && sudo mount -t nfs <IP>:/ ./target-NFS/ -o nolock
ls -l target-NFS/    # names    ;    ls -n target-NFS/   # numeric UID/GID
sudo umount ./target-NFS
```
- **Misconfig:** `rw`+`no_root_squash` → write UID-0 file (SUID `/bin/bash` escalation); `insecure` → any user mounts. Config: `/etc/exports`.
- **Gotcha:** default `root_squash` maps your root → nobody — don't waste time writing as root unless `no_root_squash` confirmed. **Always `umount`** after testing. Match a local user's UID to become the server-side user.
- **Loot:** export list, file contents, UID/GID map, SUID/backup-script escalation vector.

### 3.4 — DNS (TCP/UDP 53)
```bash
dig ns <DOMAIN> @<DNS-IP>
dig axfr <DOMAIN> @<DNS-IP>                           # zone transfer (the big win)
dig axfr internal.<DOMAIN> @<DNS-IP>                  # internal zones have looser policy
dig CH TXT version.bind <DNS-IP>                      # BIND version
dnsenum --dnsserver <DNS-IP> --enum -p 0 -s 0 -o subdomains.txt \
  -f /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt <DOMAIN>
dig soa <DOMAIN>                                      # admin email (. → @ in 1st field)
```
- **Misconfig:** `allow-transfer { any; }`, `allow-recursion { any; }`. Config: `/etc/bind/named.conf*`, `db.<domain>`.
- **Gotcha:** AXFR is **TCP/53** (filtered TCP = no AXFR even if allowed). Always specify `@<NS-IP>` from the NS list. Public AXFR fails → try `internal.`/`dev.`/`corp.`.
- **Loot:** full zone, all subdomains, NS IPs, admin email, BIND version.

### 3.5 — SMTP (TCP/25, 587, 465)
```bash
telnet <IP> 25                                       # then: EHLO x / VRFY <user> / RSET / QUIT
sudo nmap <IP> -sC -sV -p25
sudo nmap <IP> -p25 --script smtp-open-relay -v
smtp-user-enum -M VRFY -U ./wordlist.txt -t <IP>     # baseline with garbage first!
openssl s_client -connect <IP>:25 -starttls smtp     # or :465 implicit TLS
```
- **Misconfig:** `mynetworks = 0.0.0.0/0` (open relay), VRFY/EXPN enabled. Config: `/etc/postfix/main.cf`.
- **Gotcha:** useful info often only **after EHLO**. Postfix VRFY status `252` ≠ user exists — baseline with garbage. :465 = implicit TLS (`openssl s_client`); :25/:587 = STARTTLS upgrade.
- **Loot:** version/banner, valid usernames, open-relay status, cert details.

### 3.6 — IMAP / POP3 (TCP/110,143,993,995)
```bash
sudo nmap <IP> -sV -p110,143,993,995 -sC            # cert dump
openssl s_client -connect <IP>:imaps                 # or :pop3s
curl -k 'imaps://<IP>' --user <user>:<pass> -v
# IMAP: a LOGIN <u> <p> / a LIST "" * / a SELECT INBOX / a FETCH <id> (BODY[]) / a LOGOUT
# POP3: USER <u> / PASS <p> / STAT / LIST / RETR <id> / QUIT
```
- **Misconfig:** Dovecot `auth_debug_passwords = yes` → passwords in `/var/log/dovecot.log`; `auth_anonymous_username`.
- **Gotcha:** unauth `CAPABILITY` line can hide a flag/secret at the end. `-k` skips cert check (self-signed common). POP3 `DELE` permanent after `QUIT`.
- **Loot:** org/FQDN/admin-email from cert, mailbox + email contents (often **carries SSH private keys** — see lab-hard).

### 3.7 — MySQL (TCP/3306)
```bash
mysql -u root -h <IP>                                 # empty-root check (no space: -p<PASS>)
mysql -u <user> -p<password> -h <IP>
sudo nmap <IP> -sV -sC -p3306 --script mysql*         # VERIFY BY HAND
# show databases; / use <db>; / show tables; / select * from <t>; / select version();
```
- **Misconfig:** empty root pw, `bind-address = 0.0.0.0`, `secure_file_priv` empty (LOAD_FILE/INTO OUTFILE). Config: `mysqld.cnf`; creds in `mysql.user`.
- **Gotcha:** `-p<pass>` = **no space**. Always hand-verify Nmap "empty password" (false positives). MySQL 8.x may need `--ssl-mode=DISABLED`.
- **Loot:** version, user-data DBs (skip information_schema/mysql/performance_schema/sys), hashes from `mysql.user`.

### 3.8 — MSSQL (TCP/1433)
```bash
/usr/bin/impacket-mssqlclient <user>@<IP> -windows-auth
sudo nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-ntlm-info,ms-sql-config,ms-sql-tables,ms-sql-hasdbaccess,ms-sql-dump-hashes \
  --script-args mssql.instance-port=1433,mssql.username=sa,mssql.password=,mssql.instance-name=MSSQLSERVER -sV -p1433 <IP>
# in session: select name from sys.databases
```
- **Misconfig:** `sa` weak/default pw, `xp_cmdshell` (RCE), `xp_dirtree` → UNC coercion → NetNTLMv2 capture (Responder).
- **Gotcha:** `ms-sql-ntlm-info` leaks Windows computer + domain **pre-auth**. `mssql_ping` = UDP/1434 (off → must know instance+port). Encryption negotiation may fail silently — retry without `-windows-auth`.
- **Loot:** version, hostname/instance, domain, DBs, hashes (`sys.sql_logins`), RCE via `xp_cmdshell`.

### 3.9 — Oracle TNS (TCP/1521)
```bash
sudo nmap -p1521 -sV <IP> --open --script oracle-sid-brute     # SID first (lab default: XE)
python3 odat.py all -s <IP>                                     # brute SIDs + accounts (slow/noisy)
sqlplus <user>/<pass>@<IP>/<SID>                                # or: as sysdba
# select table_name from all_tables; / select name,password from sys.user$;
./odat.py utlfile -s <IP> -d <SID> -U <u> -P <p> --sysdba --putFile <dir> <name> <localfile>
```
- **Misconfig:** **SID required first**; default creds `scott/tiger`, `dbsnmp/dbsnmp`, `system/CHANGE_ON_INSTALL`. `as sysdba` escalation.
- **Gotcha:** `sys.user$` hashes = 16-hex legacy DES (check format before cracking). ODAT `all` ~5–10 min, single-IP only.
- **Loot:** SID, version, creds, tables, `sys.user$` hashes, file-write (if DBA+utlfile).

### 3.10 — SNMP (UDP/161)
```bash
snmpwalk -v2c -c public <IP> | tee SNMPWalk.txt          # try public first
onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt <IP>   # community brute
braa <community>@<IP>:.1.3.6.*                            # OID brute once community known
grep -m1 -B8 "HTB\|@inlanefreight" SNMPWalk.txt
```
- **Misconfig:** `public`/`private` strings, `rwcommunity`, `rwuser noauth`. Config: `/etc/snmp/snmpd.conf`.
- **Key OIDs:** sysContact `.1.3.6.1.2.1.1.4.0` (admin email), sysName `.1.3.6.1.2.1.1.5.0`, **process args `.1.3.6.1.2.1.25.6.3.1.2.*`** (creds from `chpasswd`/recovery scripts — see lab-hard).
- **Gotcha:** SNMPv3 ≠ v2c (silent failure). onesixtyone UDP packet loss → re-run.
- **Loot:** sysDescr, admin email, installed software (→CVE), running processes (**creds in process args**).

### 3.11 — IPMI (UDP/623)
```bash
sudo nmap -sU --script ipmi-version -p623 <IP>
# msf: use auxiliary/scanner/ipmi/ipmi_dumphashes; set rhosts <IP>; run
hashcat -m 7300 ipmi.txt /usr/share/wordlists/rockyou.txt          # 7300 = IPMI 2.0 RAKP
hashcat -m 7300 ipmi.txt -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u             # HP iLO factory mask
```
- **Misconfig:** RAKP hash extractable **pre-auth** (no login). Defaults: iDRAC `root:calvin`, Supermicro `ADMIN:ADMIN`, HP iLO `Administrator:<8-char>`.
- **Gotcha:** **mode 7300** (IPMI 2.0 RAKP) — memorize. `ipmi_dumphashes` has a built-in wordlist (auto-cracks weak ones).
- **Loot:** BMC username, RAKP hash → offline crack → full physical-equivalent control.

### 3.12 — SSH / Rsync / R-services
```bash
# SSH (22)
ssh-audit <IP>
ssh -v <user>@<IP> -o PreferredAuthentications=password
# Rsync (873)
nc -nv <IP> 873     # then: @RSYNCD: <ver>  ;  #list
rsync -av --list-only rsync://<IP>/<share>
rsync -av rsync://<IP>/<share>            # add -e "ssh -p2222" to tunnel
# R-services (512/513/514)
rlogin <IP> -l <user>            # no pw if .rhosts / hosts.equiv permits
rusers -al <IP>
```
- **Misconfig:** SSH `PermitEmptyPasswords`/`PermitRootLogin`; Rsync anonymous module exposing `.ssh/`; R-svc `.rhosts`/`hosts.equiv` with `+ +`.
- **Gotcha:** `ssh -v` listing `password` in continuable auths ≠ password auth actually allowed. Rsync anonymous shares often hold `id_rsa` → lateral move.
- **Loot:** weak-crypto/version, SSH keys + creds from Rsync shares, passwordless access via trust files.

**Output checkpoint (Phase 3):** at least one of {anonymous file, leaked cred, SSH key, hash} per accessible service. Feed all of it into Phase 4.

---

## Phase 4 — Credential-Reuse Loop (solves every lab)

**Goal:** every secret found is tested against every other service. This is the connective tissue of lab-easy/medium/hard.

**Trigger/Precondition:**

| You have | Try it against |
|---|---|
| A username:password (any source) | SMB, FTP, SSH, MySQL, MSSQL, IMAP/POP3, RDP |
| An `id_rsa` (from FTP/Rsync/email/share) | `chmod 600 id_rsa; ssh -i id_rsa <user>@<IP>` |
| A hash (RAKP / sys.user$ / mysql.user) | `hashcat` offline, then loop the cracked pw |
| Creds in a file/chat-log/ticket/process-arg | same loop — devs paste secrets everywhere |

**Lab proof:**
- **easy:** AXFR → `internal.` zone → ProFTPD on 2121 → `ceil:qwer1234` → `get id_rsa` → `ssh -i`.
- **medium:** NFS chat-log → `alex:lol123!mD` → SMB `devshare/important.txt` → `sa:87N1ns@slls83` → **password reuse** → RDP as Administrator → SSMS query.
- **hard:** SNMP community `backup` → `snmpwalk` process-args leak `tom:NMds732Js2761` → IMAPS login → SSH key in email body → SSH → MySQL same creds → query flag.

**Output checkpoint:** an interactive session (SSH/RDP) or a DB query returning the flag.

---

## Decision Tree (Under Exam Pressure)

```
Open port found?
├─ NO open ports / scan slow ──────────── re-scan: nmap -Pn -sV --top-ports 100; then -p- in background
│                                          STUCK > 5 min → check UDP (53/111/161/623); verify VPN routing
├─ FTP/SMB/NFS/Rsync open ─────────────── try anonymous/null FIRST (smbclient -N -L, ftp anon, showmount -e)
│   ├─ anon works → loot files; grep for id_rsa / creds / .conf
│   └─ anon denied → rpcclient null + RID-brute (SMB); STUCK > 10 min → move to next service, come back
├─ DNS (53) open ──────────────────────── dig axfr @<NS>; fails → dig axfr internal.<domain> @<NS>
│   └─ all AXFR denied → dnsenum 110k wordlist; STUCK > 10 min → subdomain-brute, pivot to other ports
├─ SMTP (25) open ─────────────────────── EHLO; VRFY (baseline garbage first); smtp-user-enum
│   └─ VRFY disabled → note version/banner, move on (don't burn the connection)
├─ SNMP (161/udp) open ────────────────── snmpwalk -c public; fails → onesixtyone brute
│   └─ community found → snmpwalk | grep process-args + @<domain> (creds live in hrSWRunPerf)
├─ DB open (3306/1433/1521) ───────────── empty/default creds first; VERIFY nmap by hand
│   └─ no creds yet → park it; revisit after Phase 4 finds a password to reuse
├─ Found a credential anywhere ────────── PHASE 4 LOOP: test it on EVERY other open service
└─ Found id_rsa ──────────────────────── chmod 600; ssh -i id_rsa <user>@<IP> (user often = banner/file owner)

Whole-box STUCK > 20 min → list every open port, run its Phase-3 anon/default-cred line once more,
then grep all looted files for: id_rsa, password, .htb, BEGIN PRIVATE KEY, @<domain>
```

No dead-ends: every branch ends in a next action or an explicit pivot.

---

## Signal → Counter-Move Reference

| Signal / Symptom | Likely Cause | Exact Counter-Move |
|---|---|---|
| `dig any` returns only HINFO | RFC 8482 filtering | Query types individually: `dig <d> A/MX/TXT/NS`; `dig soa` for admin email |
| AXFR refused on apex zone | `allow-transfer` locked | `dig axfr internal.<domain> @<NS>` — internal zones laxer; else `dnsenum` 110k |
| SMB "login successful" but `ls` errors | Authed to IPC$, not the share | `rpcclient -U "" <IP>` + `netshareenumall`; RID-brute for users |
| `enumdomusers` ACCESS_DENIED | Enum locked, queryuser open | RID-brute loop `seq 500 1100` → `queryuser 0x<hex>` |
| `mount` → permission denied | Export scoped to subnet / needs priv port | Re-check `showmount -e`; ensure `-o nolock`; try from in-scope source |
| NFS write as root fails / chmod denied | Default `root_squash` | Confirm `no_root_squash` before trying; else match local UID to server UID |
| Nmap says MySQL empty password | NSE false positive | Hand-verify: `mysql -u root -h <IP>` (no `-p`) |
| `mysql -u root -p P4SS` rejected | Space after `-p` | `mysql -u root -pP4SS -h <IP>` (no space) |
| SMTP VRFY returns 252 for everything | Postfix default behaviour | Baseline with garbage user; only treat as signal if real ≠ garbage |
| SNMP v2c query times out silently | Daemon is SNMPv3-only | No v2c path — note and move on |
| onesixtyone finds nothing | UDP packet loss | Re-run; add `-w 100` on lossy links |
| `ssh -v` lists password auth but login fails | `PasswordAuthentication no` still applies | Continuable-auths list ≠ enabled; need a key/other method |
| IMAP/POP3 looks empty pre-login | Pre-login CAPABILITY set | `openssl s_client` → read full CAPABILITY line (flag often appended) |
| Have a password, current service is hardened | Wrong target for this cred | **Phase 4** — reuse it on SMB/SSH/RDP/DB/IMAP |
| ProFTPD/SSH banner names a user | Banner = username hint | Try that exact username (`ceil`, etc.) for login + key file |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# Subdomains from CT logs
curl -s "https://crt.sh/?q=$DOMAIN&output=json" | jq -r '.[].name_value' | sort -u

# Full TCP + top UDP triage
sudo nmap -sV -sC -p- $IP --open && sudo nmap -sU -sV -p U:53,111,161,623 $IP

# Anonymous everything (run all, see what bites)
ftp $IP ; smbclient -N -L //$IP ; showmount -e $IP ; nc -nv $IP 873

# DNS zone transfer (apex then internal)
dig axfr $DOMAIN @$DNS_IP ; dig axfr internal.$DOMAIN @$DNS_IP

# SMB null-session full recon + RID brute
enum4linux-ng -A $IP
for i in $(seq 500 1100); do rpcclient -N -U "" $IP -c "queryuser 0x$(printf '%x\n' $i)" 2>/dev/null | grep "User Name"; done

# NFS mount/loot/unmount
mkdir nfs && sudo mount -t nfs $IP:/ ./nfs -o nolock ; ls -lA nfs/ ; sudo umount ./nfs

# SMTP user enum (baseline garbage first)
smtp-user-enum -M VRFY -U users.txt -t $IP

# SNMP — community then creds in process args
onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt $IP
snmpwalk -v2c -c $COMMUNITY $IP | tee snmp.txt && grep -E "HTB\{|@$DOMAIN|chpasswd" snmp.txt

# IPMI hash → crack
msfconsole -q -x "use auxiliary/scanner/ipmi/ipmi_dumphashes; set rhosts $IP; run; exit"
hashcat -m 7300 ipmi.txt /usr/share/wordlists/rockyou.txt

# Oracle TNS — SID then default creds
sudo nmap -p1521 -sV $IP --script oracle-sid-brute
sqlplus scott/tiger@$IP/$SID as sysdba

# Credential reuse loop (the lab solver)
smbclient -L //$IP -U $USER%$PASS ; ssh $USER@$IP ; mysql -u $USER -p$PASS -h $IP
chmod 600 id_rsa && ssh -i id_rsa $USER@$IP
```

---

## Quick Reference — Tools by Function

| Function | Tools |
|---|---|
| Subdomain / passive | `curl crt.sh`, `dig`, `host`, `shodan`, `dnsenum` |
| Port scan | `nmap -sV -sC -p-` (TCP), `nmap -sU` (UDP) |
| FTP | `ftp`, `wget -m`, `nc`, `openssl s_client -starttls ftp` |
| SMB | `smbclient`, `smbmap`, `crackmapexec`, `enum4linux-ng`, `rpcclient`, `samrdump.py` |
| NFS | `showmount`, `mount -t nfs`, `nmap --script nfs*` |
| DNS | `dig axfr`, `dnsenum`, `dig CH TXT version.bind` |
| SMTP | `telnet`, `smtp-user-enum`, `nmap --script smtp-open-relay`, `openssl s_client -starttls smtp` |
| IMAP/POP3 | `openssl s_client`, `curl -k imaps://`, `nmap -sC` (cert) |
| SNMP | `snmpwalk`, `onesixtyone`, `braa` |
| MySQL | `mysql`, `nmap --script mysql*` |
| MSSQL | `impacket-mssqlclient`, `nmap --script ms-sql-*` |
| Oracle | `nmap --script oracle-sid-brute`, `odat.py`, `sqlplus` |
| IPMI | `nmap --script ipmi-version`, `msf ipmi_dumphashes`, `hashcat -m 7300` |
| SSH/Rsync/R | `ssh-audit`, `rsync`, `rlogin`, `rusers` |

---

## Top Gotchas

1. **`-p<pass>` no space** — `mysql -u root -pP4SS` works; `-p P4SS` does not.
2. **Nmap empty-password / brute scripts false-positive** — always hand-verify MySQL/MSSQL "empty password".
3. **AXFR is TCP/53** — filtered TCP = no zone transfer even when `allow-transfer { any; }`.
4. **`dig any` is dead** (RFC 8482) — query record types individually or you'll think DNS is barren.
5. **NFS default `root_squash`** maps your root → nobody — confirm `no_root_squash` before trying SUID writes; `umount` every test.
6. **SMB: "login successful" ≠ share access** — that's IPC$; use `rpcclient` + RID-brute when `enumdomusers` is denied.
7. **SNMP creds hide in process args** (`hrSWRunPerf`, `.1.3.6.1.2.1.25.6.3.1.2.*`) — `chpasswd`/recovery scripts log the password there; `grep chpasswd snmp.txt`.
8. **IMAP carries SSH keys** — `FETCH 1 (BODY[])` can contain a whole `BEGIN OPENSSH PRIVATE KEY`; strip the SMTP/MIME junk, keep only the key block.
9. **IPMI = hashcat mode 7300** (RAKP 2.0); hash is extractable pre-auth — no login needed.
10. **SMTP VRFY 252 ≠ user exists** on Postfix — baseline with a garbage username first.
11. **Non-standard ports** — FTP on 2121, SSH on 2222 etc.; cross-reference DNS/banner, don't assume defaults.
12. **`chmod 600 id_rsa` is mandatory** — SSH silently refuses world-readable keys; the username is usually the file owner / banner name.
13. **Banner = username hint** — ProFTPD `(Ceil's FTP)` → user `ceil`. Read banners literally.
14. **Test password reuse against ALL services** — a leaked `sa:` cred is not just for MSSQL (lab-medium → Administrator over RDP).

---

## Related Vault Notes

- Framework / theory: [[00-overview]] · [[01-enumeration-principles]] · [[02-enumeration-methodology]]
- Passive recon: [[03-domain-information]] · [[04-cloud-resources]] · [[05-staff]]
- Services: [[06-ftp]] · [[07-smb]] · [[08-nfs]] · [[09-dns]] · [[10-smtp]] · [[11-imap-pop3]] · [[12-snmp]] · [[13-mysql]] · [[14-mssql]] · [[15-oracle-tns]] · [[16-ipmi]] · [[17-linux-remote-mgmt]]
- Labs: [[19-lab-easy]] · [[20-lab-medium]] · [[21-lab-hard]]
- Service exploitation (next step): [[../common-services/00-METHODOLOGY]]
- Credential cracking/reuse: [[../password-attacks/00-METHODOLOGY]] · [[../login-brute-forcing/00-METHODOLOGY]]
- Port scanning depth: [[../nmap/00-overview]]

Triage by symptom: [[../ATTACK-PATHS]]
