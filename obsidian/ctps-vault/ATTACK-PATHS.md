# CPTS — Attack-Path Triage Map

**Purpose:** symptom / state → which note to open. Use this when you know *what you have* but not *what to try next*.
**Updated:** 2026-05-04

---

## 1. Just got a target — recon phase

- **External, no creds, only a domain** → `ad-enum-attacks/04-external-recon.md`
- **Internal, on the network, nothing else** → `ad-enum-attacks/05-initial-domain-enum.md` + `ad-enum-attacks/06-llmnr-nbtns-poisoning-linux.md`
- **Need to interact with random services** → `common-services/01-interacting-with-services.md`

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
| SQL error / weird response on quote | `sql-injection-fundamentals/08-intro-to-sqli.md` → `09-subverting-query-logic.md` → `11-union-clause.md` → `12-union-injection.md` |
| Confirmed SQLi, want fast pwn | `sqlmap-fundamentals/04-http-request.md`, `10-os-exploitation.md` |
| SQLi behind WAF / CSRF token | `sqlmap-fundamentals/09-bypassing-protections.md` |
| Need to fuzz hidden params/dirs | `file-inclusion/09-automated-scanning.md`, `web-proxies/10-burp-intruder.md`, `web-proxies/11-zap-fuzzer.md` |
| Need to manipulate raw request | `web-proxies/04-intercepting-requests.md`, `07-repeating-requests.md`, `08-encoding-decoding.md` |

## 4. AD — by what you currently have

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

## 5. Pivoting / Tunneling

| Need | Note |
|------|------|
| Local port forward via SSH | `pivoting-tunneling/05-ssh-port-forwarding.md` |
| SOCKS proxy via SSH (most common) | `pivoting-tunneling/05-ssh-port-forwarding.md` (dynamic forward + proxychains) |
| Pivot from a Windows host | `pivoting-tunneling/09-plink-windows.md` |
| VPN-like tunnel over SSH | `pivoting-tunneling/10-sshuttle.md` |
| Pivot via Meterpreter | `pivoting-tunneling/06-meterpreter-port-forwarding.md` |
| Reverse shell through firewall via socat | `pivoting-tunneling/07-socat-redirection.md` |
| Pivot through DMZ web server | `pivoting-tunneling/11-rpivot.md` |
| Only ICMP outbound | `pivoting-tunneling/13-ptunnel-icmp.md` |
| Multi-hop to DC (skills assessment style) | `pivoting-tunneling/skills-assessment.md` |

## 6. Credential cracking / brute force

| Have | Next |
|------|------|
| NTLMv2 hash from Responder | hashcat `-m 5600` (see `ad-enum-attacks/06`/`07`) |
| Kerberoast TGS hash | hashcat `-m 13100` (see `ad-enum-attacks/17`) |
| Need to spray known cred across services | `login-brute-forcing/06-hydra.md`, `09-medusa.md` |
| Need to filter wordlist before attack | `login-brute-forcing/05-hybrid-attacks.md` |

## 7. Reporting (when the engagement is "done")

- Methodology overview → `documentation/01-intro-to-documentation-reporting.md`
- Note structure & evidence handling → `documentation/02-notetaking-and-organization.md`
- Report types & expectations → `documentation/03-types-of-reports.md`
- Required sections → `documentation/04-components-of-a-report.md`
- Writing a finding → `documentation/05-how-to-write-up-a-finding.md`
- QA / polish / templates → `documentation/06-reporting-tips-and-tricks.md`

## 8. Gaps in the vault (don't waste time searching here — go to HTB notes)

- `nmap/`, `footprinting/`, `ffuf/`, `web-recon/` — empty
- `shells-payloads/` — empty (need for revshells, msfvenom, payload formats)
- `linux-privesc/`, `windows-privesc/` — empty
- `password-attacks/` — only overview stub (john/hashcat/SAM/LSASS/PtH/PtT detail missing)
- `ad-enum-attacks/` — sections 26, 28, 30, 33, 34, 35 missing

## How to use this with Claude (cheapest path)

1. Tell Claude the **state/symptom** ("I have an NTLM hash for a regular user", "port 88 is open", "found a `?page=` parameter").
2. Claude greps `ATTACK-PATHS.md` and `SEARCH.md` first → resolves to a file path.
3. Claude `Read`s **only** the matching note(s). No vault-wide scanning.
