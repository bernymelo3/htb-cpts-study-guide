# CPTS Vault — Search Index

**Purpose:** flat keyword index for fast lookup. Grep this file FIRST, then Read the matching note.
**Format:** `module/file.md | Title | tags`
**Updated:** 2026-05-07 (login-brute-forcing methodology added)

---

AD-PRESENTATION-SLIDES.md | <untagged> | -
AD-PRESENTATION.md | <untagged> | -
AD-SIMPLE-GUIDE.md | <untagged> | -
ATTACK-PATHS.md | <untagged> | -
INDEX.md | <untagged> | -
METHODOLOGY-REGISTRY.md | <untagged> | -
SEARCH.md | <untagged> | -
_templates/methodology-prompt.md | <untagged> | -
ad-enum-attacks/00-overview.md | <untagged> | -
ad-enum-attacks/01-intro-to-ad.md | Section 1 — Introduction to Active Directory Enumeration & Attacks | active-directory, enumeration, theory, introduction, ad-attacks
ad-enum-attacks/02-tools-of-the-trade.md | Section 2 — Tools of the Trade | tools, reference, enumeration, attacks, active-directory
ad-enum-attacks/03-scenario.md | Section 3 — Scenario | scenario, scope, rules-of-engagement, theory, engagement
ad-enum-attacks/04-external-recon.md | Section 4 — External Recon and Enumeration Principles | external-recon, osint, enumeration, theory, dns
ad-enum-attacks/05-initial-domain-enum.md | Section 5 — Initial Enumeration of the Domain | enumeration, ad-enumeration, nmap, kerbrute, wireshark, host-discovery
ad-enum-attacks/06-llmnr-nbtns-poisoning-linux.md | <untagged> | -
ad-enum-attacks/07-llmnr-nbtns-poisoning-windows.md | <untagged> | -
ad-enum-attacks/08-password-spraying-overview.md | Section 8 — Password Spraying Overview | password-spraying, active-directory, kerbrute, enumeration, theory, osint
ad-enum-attacks/09-enumerating-password-policies.md | Section 9 — Enumerating & Retrieving Password Policies | active-directory, password-policy, smb-null-session, ldap-anonymous-bind, enum4linux, password-spraying
ad-enum-attacks/10-password-spraying-user-list.md | Section 10 — Password Spraying - Making a Target User List | active-directory, user-enumeration, kerbrute, password-spraying, smb-null-session, ldap-anonymous-bind
ad-enum-attacks/11-internal-password-spraying-linux.md | Section 11 — Internal Password Spraying - from Linux | password-spraying, crackmapexec, kerbrute, rpcclient, active-directory, lateral-movement
ad-enum-attacks/12-internal-password-spraying-windows.md | Section 12 — Internal Password Spraying - from Windows | password-spraying, domainpasswordspray, powershell, active-directory, windows
ad-enum-attacks/13-enumerating-security-controls.md | Section 13 — Enumerating Security Controls | defender, applocker, laps, powershell, enumeration, security-controls
ad-enum-attacks/14-credentialed-enum-linux.md | Section 14 — Credentialed Enumeration - from Linux | active-directory, crackmapexec, smbmap, rpcclient, impacket, bloodhound, ldap
ad-enum-attacks/15-credentialed-enum-windows.md | Section 15 — Credentialed Enumeration - from Windows | active-directory, powerview, bloodhound, sharphound, snaffler, windows-enumeration
ad-enum-attacks/16-living-off-the-land.md | Section 16 — Living Off the Land | active-directory, living-off-the-land, lotl, dsquery, wmi, net-commands
ad-enum-attacks/17-kerberoasting-linux.md | Section 17 — Kerberoasting from Linux | kerberoasting, spn, impacket, hashcat, lateral-movement, privilege-escalation
ad-enum-attacks/18-kerberoasting-windows.md | Section 18 — Kerberoasting from Windows | kerberoasting, spn, rubeus, powerview, mimikatz, windows
ad-enum-attacks/19-acl-abuse-primer.md | Section 19 — Access Control List (ACL) Abuse Primer | acl, dacl, ace, genericall, genericwrite, privilege-escalation
ad-enum-attacks/20-acl-enumeration.md | Section 20 — ACL Enumeration | acl, enumeration, powerview, bloodhound, dacl, ace
ad-enum-attacks/21-acl-abuse-tactics.md | Section 21 — ACL Abuse Tactics | acl-abuse, forcechangepassword, genericwrite, genericall, targeted-kerberoasting, cleanup
ad-enum-attacks/22-dcsync.md | Section 22 — DCSync | dcsync, active directory, secretsdump, mimikatz, ntds, credential dumping
ad-enum-attacks/23-privileged-access.md | Section 23 — Privileged Access | lateral-movement, rdp, winrm, sqladmin, bloodhound, evil-winrm
ad-enum-attacks/24-winrm-double-hop-kerberos.md | <untagged> | -
ad-enum-attacks/25-bleeding-edge-vulnerabilities.md | Section 25 — Bleeding Edge Vulnerabilities | nopac, printnightmare, petitpotam, privilege-escalation, domain-compromise, ad-cs
ad-enum-attacks/27-domain-trusts-primer.md | Section 27 — Domain Trusts Primer | active-directory, domain-trusts, enumeration, powerview, bloodhound, netdom
ad-enum-attacks/29-domain-trusts-child-to-parent-linux.md | Section 29 — Attacking Domain Trusts - Child -> Parent Trusts - from Linux | active-directory, domain-trusts, extrasids, impacket, golden-ticket, linux
ad-enum-attacks/31-cross-forest-trust-abuse-linux.md | Section 31 — Cross-Forest Trust Abuse - from Linux | kerberoasting, cross-forest, impacket, bloodhound-python, linux, getuserspns
ad-enum-attacks/32-hardening-active-directory.md | Section 32 — Hardening Active Directory | defense, hardening, mitre, protected-users, policy
ad-enum-attacks/36-beyond-this-module.md | Section 36 — Beyond this Module | resources, practice, learning-path, community
ad-enum-attacks/skills-assessment-part1.md | <untagged> | -
ad-enum-attacks/skills-assessment-part2.md | Skills Assessment Part II — Full-Scope Internal AD Compromise | responder, password-spraying, acl-abuse, genericall, dcsync, getsystem
command-injetions/01-intro.md | Section 1 — Intro to Command Injections | theory, concept, command-injection, injection-types, owasp
common-services/00-overview.md | <untagged> | -
common-services/01-interacting-with-services.md | Section 1 — Interacting with Common Services | theory, methodology, enumeration, smb, databases, email
common-services/02-attack-concept.md | Section 2 — The Concept of Attacks | theory, methodology, vulnerability-analysis, log4j, concept
common-services/03-service-misconfigurations.md | Section 3 — Service Misconfigurations | theory, misconfiguration, default-credentials, access-control, enumeration
common-services/04-finding-sensitive-info.md | Section 4 — Finding Sensitive Information | theory, enumeration, data-leakage, pivoting, credential-hunting
common-services/06-ftp-latest-vulns.md | Section 6 — Latest FTP Vulnerabilities | theory, vulnerability, ftp, directory-traversal, cve, coreftp
common-services/07-attacking-smb.md | Section 7 — Attacking SMB | smb, enumeration, null-session, brute-force, ssh, private-key, rpc, crackmapexec
common-services/08-smb-latest-vulns.md | Section 8 — Latest SMB Vulnerabilities | theory, vulnerability, smb, rce, integer-overflow, cve
common-services/11-attacking-rdp.md | Section 11 — Attacking RDP | rdp, pass-the-hash, ntlm, xfreerdp, registry, restricted-admin
common-services/13-attacking-dns.md | Section 13 — Attacking DNS | dns, enumeration, zone-transfer, axfr, subdomain, subbrute, txt-record
common-services/14-dns-latest-vulns.md | Section 14 — Latest DNS Vulnerabilities | theory, dns, subdomain-takeover, cname, misconfiguration, bug-bounty
common-services/15-attacking-email-services.md | Section 15 — Attacking Email Services | smtp, pop3, email, enumeration, brute-force, open-relay, swaks
common-services/16-email-latest-vulns.md | Section 16 — Latest Email Service Vulnerabilities | theory, vulnerability, smtp, rce, opensmtpd, cve
common-services/17-skills-assessment-easy.md | Skills Assessment — Easy (WIN-EASY) | skills-assessment, smtp, mysql, credential-reuse, file-read, flag
common-services/19-skills-assessment-hard.md | Skills Assessment — Hard | skills-assessment, smb, rdp, mssql, impersonation, linked-server, privilege-escalation
documentation/00-overview.md | <untagged> | -
documentation/01-intro-to-documentation-reporting.md | Section 1 — Introduction to Documentation and Reporting | theory, documentation, reporting, methodology, notetaking, soft-skills
documentation/02-notetaking-and-organization.md | Section 2 — Notetaking & Organization | documentation, reporting, tmux, organization, evidence, notetaking
documentation/03-types-of-reports.md | Section 3 — Types of Reports | documentation, reporting, assessment-types, penetration-testing, methodology
documentation/04-components-of-a-report.md | Section 4 — Components of a Report | documentation, reporting, executive-summary, attack-chain, findings, active-directory
documentation/05-how-to-write-up-a-finding.md | Section 5 — How to Write Up a Finding | reporting, findings, documentation, remediation, writehat, evidence, cvss
documentation/06-reporting-tips-and-tricks.md | Section 6 — Reporting Tips and Tricks | reporting, documentation, ms-word, client-communication, QA, templates, findings-database
documentation/07-section-doc-reporting-lab.md | Skills Assessment — Documentation & Reporting Practice Lab | active-directory, responder, ntlm, hashcat, crackmapexec, domain-admin
file-inclusion/00-METHODOLOGY.md | File Inclusion (LFI/RFI) — Full Pentest Methodology | methodology, lfi, rfi, file-inclusion, php-wrappers, log-poisoning, exam, cheatsheet, decision-tree
file-inclusion/00-overview.md | <untagged> | -
file-inclusion/01-intro-to-file-inclusion.md | Section 1 — Intro to File Inclusions | theory, file-inclusion, lfi, php, nodejs, java, dotnet
file-inclusion/02-lfi-basics.md | Section 2 — Local File Inclusion (LFI) | lfi, file-inclusion, path-traversal, php, web
file-inclusion/03-basic-bypasses.md | Section 3 — Basic Bypasses | lfi, file-inclusion, path-traversal, filter-bypass, php
file-inclusion/04-php-filters.md | Section 4 — PHP Filters (Source Code Disclosure via php://filter) | lfi, php-wrappers, php-filter, base64, source-disclosure
file-inclusion/05-php-wrappers.md | Section 5 — PHP Wrappers (RCE via data://, php://input, expect://) | lfi, rce, php-wrappers, data-wrapper, webshell, php
file-inclusion/06-rfi.md | Section 6 — Remote File Inclusion (RFI) | lfi, rfi, rce, php, webshell, ssrf
file-inclusion/07-lfi-and-file-uploads.md | Section 7 — LFI and File Uploads | lfi, rce, file-upload, php, webshell, phar, zip
file-inclusion/08-log-poisoning.md | Section 8 — Log Poisoning | lfi, rce, log-poisoning, session-poisoning, apache, php
file-inclusion/09-automated-scanning.md | Section 9 — Automated Scanning | lfi, ffuf, fuzzing, path-traversal, parameter-discovery
file-inclusion/10-prevention.md | Section 10 — File Inclusion Prevention | lfi, prevention, php.ini, hardening, disable-functions, apache
file-inclusion/skills-assessment.md | Skills Assessment — File Inclusion | lfi, file-upload, path-traversal, double-url-encoding, rce
file-uploads/00-METHODOLOGY.md | File Upload Attacks — Full Pentest Methodology | methodology, file-upload, exam, cheatsheet, decision-tree, web-shell, xxe, svg, phar, burp
file-uploads/01 - File Uploads.md | Section 1 — Intro to File Upload Attacks | theory, concept, file-upload, web-attacks, vulnerability
file-uploads/02-absent-validation.md | Section 2 — Absent Validation | file-upload, rce, web-shell, php, arbitrary-upload
file-uploads/03-upload-exploitation.md | Section 3 — Upload Exploitation | file-upload, web-shell, reverse-shell, msfvenom, rce, php
file-uploads/04-file_upload_attacks_client_side_validation.md | Section 4 — Client-Side Validation | file-upload, client-side-bypass, web-shell, burp-suite, php, javascript
file-uploads/05-file_upload_attacks_blacklist_filters.md | Section 5 — Blacklist Filters | file-upload, blacklist-bypass, php-extensions, fuzzing, burp-intruder
file-uploads/06-whitelist-filters.md | Section 6 — Whitelist Filters | file-upload, whitelist-bypass, reverse-double-extension, apache-misconfiguration, php, burp-intruder
file-uploads/07-type-filters.md | Section 7 — Type Filters | file-upload, content-type, mime-type, magic-bytes, gif8, burp-repeater
file-uploads/08-limited-file-uploads.md | Section 8 — Limited File Uploads | xxe, svg, file-upload, xss, dos, php-filter
file-uploads/09-other-upload-attacks.md | Section 9 — Other Upload Attacks | file-upload, filename-injection, directory-disclosure, windows-quirks, advanced-attacks
file-uploads/11-skills-assessment-file-upload-attacks.md | Skills Assessment — File Upload Attacks | file-upload, xxe, svg, phar, burp-intruder, rce, whitebox
login-brute-forcing/00-METHODOLOGY.md | Login Brute Forcing — Full Pentest Methodology | methodology, brute-forcing, login-attacks, hydra, medusa, password-spraying, exam, cheatsheet, decision-tree, credential-stuffing
login-brute-forcing/00-overview.md | <untagged> | -
login-brute-forcing/01-intro.md | Section 1 — Login Brute Forcing | brute-forcing, password-attacks, theory, methodology, authentication
login-brute-forcing/02-password-security.md | Section 2 — Password Security Fundamentals | password-security, theory, defense, best-practices, default-credentials
login-brute-forcing/03-pin-cracking.md | Section 3 — Brute Force Attacks (PIN Cracking) | brute-forcing, pin-cracking, python, web-attack, automation
login-brute-forcing/04-dictionary-attacks.md | Section 4 — Dictionary Attacks (Wordlist Password Cracking) | dictionary-attack, wordlist, python, seclists, credential-guessing
login-brute-forcing/05-hybrid-attacks.md | Section 5 — Hybrid Attacks (Wordlist Filtering & Credential Stuffing) | hybrid-attacks, wordlist-filtering, grep, regex, credential-stuffing
login-brute-forcing/06-hydra.md | Section 6 — Hydra (Network Login Cracker) | hydra, brute-forcing, password-attacks, tool, network-cracker
login-brute-forcing/07-basic-http-auth.md | Section 7 — Basic HTTP Authentication | hydra, http-basic-auth, brute-forcing, web-attacks
login-brute-forcing/08-hydra-http-post-form.md | Section 8 — Login Forms (http-post-form with Hydra) | hydra, http-post-form, web-forms, brute-forcing, login-form
login-brute-forcing/09-medusa.md | Section 9 — Medusa (Parallel Login Brute‑Forcer) | medusa, brute-forcing, password-attacks, tool, parallel-attacks
login-brute-forcing/10-ssh-ftp-brute.md | Section 10 — Web Services (SSH + FTP Brute Forcing) | medusa, ssh, ftp, pivoting, brute-forcing, internal-enumeration
login-brute-forcing/11-module-summary.md | Module Summary — Login Brute Forcing (Full Reference) | summary, reference, hydra, medusa, wordlists, skills-assessment
password-attacks/00-overview.md | <untagged> | -
pivoting-tunneling/00-overview.md | <untagged> | -
pivoting-tunneling/01-intro.md | Section 1 — Introduction to Pivoting, Tunneling, and Port Forwarding | pivoting, tunneling, port-forwarding, lateral-movement, theory, beginner
pivoting-tunneling/02-networking-basics.md | Section 2 — The Networking Behind Pivoting | networking, theory, ip-addressing, routing, pivoting, beginner
pivoting-tunneling/05-ssh-port-forwarding.md | Section 5 — SSH Port Forwarding: Local, Dynamic, and Remote | ssh, port-forwarding, socks, proxychains, reverse-shell, pivoting
pivoting-tunneling/06-meterpreter-port-forwarding.md | <untagged> | -
pivoting-tunneling/07-socat-redirection.md | Section 7 — Socat Redirection with a Reverse Shell | socat, pivoting, reverse-shell, redirection, port-forwarding
pivoting-tunneling/09-plink-windows.md | Section 9 — SSH for Windows: plink.exe | plink, windows, ssh, tunneling, pivoting, socks-proxy
pivoting-tunneling/10-sshuttle.md | Section 10 — SSH Pivoting with Sshuttle | sshuttle, pivoting, tunneling, vpn, ssh
pivoting-tunneling/11-rpivot.md | Section 11 — Web Server Pivoting with Rpivot | rpivot, socks-proxy, reverse-proxy, pivoting, python2
pivoting-tunneling/13-ptunnel-icmp.md | Section 13 — ICMP Tunneling with ptunnel‑ng | icmp-tunneling, ptunnel-ng, pivoting, firewall-evasion
pivoting-tunneling/17-detection-prevention.md | Section 17 — Detection & Prevention | detection, prevention, mitre, defender, blue-team, theory
pivoting-tunneling/skills-assessment.md | Skills Assessment — Multi‑Hop Pivot to Domain Controller | skills-assessment, pivoting, rdp, mimikatz, ssh-tunnel, proxychains
sql-injection-fundamentals/00-overview.md | <untagged> | -
sql-injection-fundamentals/01-intro.md | Section 1 — Introduction | theory, sql-injection, web-app, introduction, mysql
sql-injection-fundamentals/02-intro-to-databases.md | Section 2 — Intro to Databases | theory, databases, dbms, architecture, mysql, introduction
sql-injection-fundamentals/03-types-of-databases.md | Section 3 — Types of Databases | theory, databases, relational-database, nosql, mysql, mongodb
sql-injection-fundamentals/04-intro-to-mysql.md | Section 4 — Intro to MySQL | mysql, sql, database, create-table, mariadb, cli
sql-injection-fundamentals/05-sql-statements.md | Section 5 — SQL Statements | mysql, sql, insert, select, update, alter, drop
sql-injection-fundamentals/06-query-results.md | Section 6 — Query Results | mysql, sql, order-by, limit, where, like, wildcard
sql-injection-fundamentals/07-sql-operators.md | Section 7 — SQL Operators | mysql, sql, operators, and, or, not, precedence
sql-injection-fundamentals/08-intro-to-sqli.md | Section 8 — Intro to SQL Injections | theory, sql-injection, union-based, error-based, blind-sqli, injection-types
sql-injection-fundamentals/09-subverting-query-logic.md | Section 9 — Subverting Query Logic | sqli, authentication-bypass, or-injection, operator-precedence, login-bypass
sql-injection-fundamentals/10-using-comments.md | Section 10 — Using Comments | sqli, comments, authentication-bypass, parentheses-escape, comment-injection
sql-injection-fundamentals/11-union-clause.md | Section 11 — Union Clause | sqli, union, column-matching, junk-data, union-clause
sql-injection-fundamentals/12-union-injection.md | Section 12 — Union Injection | sqli, union-injection, order-by, column-detection, data-extraction
sql-injection-fundamentals/13-database-enumeration.md | Section 13 — Database Enumeration | sqli, union-injection, information-schema, database-enumeration, data-extraction
sql-injection-fundamentals/15-writing-files.md | Section 15 — Writing Files | sqli, file-write, webshell, into-outfile, rce, secure-file-priv
sql-injection-fundamentals/16-mitigation.md | Section 16 — Mitigating SQL Injection | theory, mitigation, defence, sql-injection, parameterized-queries, waf
sql-injection-fundamentals/skills-assessment.md | Skills Assessment — SQL Injection Fundamentals | sqli, skills-assessment, union-injection, webshell, rce, nginx
sqlmap-fundamentals/00-METHODOLOGY.md | SQLMap Essentials — Full Pentest Methodology | methodology, sqlmap, sqli, exam, cheatsheet, decision-tree, enumeration, tamper, os-shell, csrf
sqlmap-fundamentals/00-overview.md | <untagged> | -
sqlmap-fundamentals/01-overview.md | Section 1 — SQLMap Overview | sqlmap, sql-injection, tool-overview, theory, enumeration, exploitation
sqlmap-fundamentals/02-getting-started.md | Section 2 — Getting Started with SQLMap | sqlmap, help, usage, example, command-line
sqlmap-fundamentals/03-output-description.md | Section 3 — SQLMap Output Description | sqlmap, output, interpretation, theory, scan, log
sqlmap-fundamentals/04-http-request.md | Section 4 — Running SQLMap on an HTTP Request | sqlmap, sqli, post-data, cookie-injection, json-injection, burp-request
sqlmap-fundamentals/05-handling-errors.md | Section 5 — Handling SQLMap Errors | sqlmap, errors, debugging, verbose, proxy, traffic
sqlmap-fundamentals/06-attack-tuning.md | Section 6 — Attack Tuning | sqlmap, sqli, level-risk, prefix-suffix, union-tuning, attack-tuning
sqlmap-fundamentals/07-database-enumeration.md | Section 7 — Database Enumeration | sqlmap, sqli, enumeration, dump, database, tables
sqlmap-fundamentals/08-advanced-enumeration.md | Section 8 — Advanced Database Enumeration | sqlmap, sqli, schema, search, password-cracking, hash
sqlmap-fundamentals/09-bypassing-protections.md | Section 9 — Bypassing Web Application Protections | sqlmap, sqli, csrf-bypass, waf-bypass, tamper-scripts, user-agent
sqlmap-fundamentals/10-os-exploitation.md | Section 10 — OS Exploitation | sqlmap, sqli, file-read, file-write, os-shell, webshell
web-proxies/00-overview.md | <untagged> | -
web-proxies/01-intro.md | Section 1 — Intro to Web Proxies | theory, web-proxy, burp-suite, zap, introduction, concept
web-proxies/02-installation.md | <untagged> | -
web-proxies/03-proxy-setup.md | Section 3 — Proxy Setup | theory, proxy-setup, burp-suite, zap, foxyproxy, ca-certificate
web-proxies/04-intercepting-requests.md | Section 4 — Intercepting Web Requests | burp suite, zap, proxy, intercept, command injection, request manipulation
web-proxies/05-intercepting-responses.md | Section 5 — Intercepting Responses | intercept, response-modification, burp-suite, zap, hud, client-side
web-proxies/06-automatic-modification.md | Section 6 — Automatic Modification | burp-suite, zap, match-and-replace, automation, proxy, header-modification
web-proxies/06-zap-replacer-notes.md | <untagged> | -
web-proxies/07-repeating-requests.md | Section 7 — Repeating Requests | burp, zap, repeater, request-repeating, command-injection, proxy
web-proxies/08-encoding-decoding.md | Section 8 — Encoding/Decoding | encoding, decoding, burp-suite, zap, base64, url-encoding
web-proxies/09-proxying-tools.md | Section 9 — Proxying Tools | burp, zap, proxychains, metasploit, proxy, cli-tools
web-proxies/10-burp-intruder.md | Section 10 — Burp Intruder | burp, intruder, fuzzing, brute-force, wordlists, web-proxy
web-proxies/11-zap-fuzzer.md | Section 11 — ZAP Fuzzer | zap, fuzzer, fuzzing, cookies, md5, wordlists
web-proxies/12-burp-scanner.md | Section 12 — Burp Scanner | burp-suite, scanner, pro-feature, crawl, active-scan, passive-scan
web-proxies/13-zap-scanner.md | Section 13 — ZAP Scanner | zap, scanner, spider, active-scan, passive-scan, command-injection
web-proxies/14-extensions.md | Section 14 — Extensions | extensions, burp-suite, zap, bapp-store, marketplace, plugins
web-proxies/skills-assessment.md | Skills Assessment — Using Web Proxies | zap, burp-suite, intruder, cookie-decoding, proxy, metasploit
xss/00-METHODOLOGY.md | Cross-Site Scripting (XSS) — Full Pentest Methodology | methodology, xss, exam, cheatsheet, decision-tree, stored-xss, reflected-xss, dom-xss, phishing, xsstrike
xss/00-overview.md | <untagged> | -
xss/01-intro.md | Section 1 — Introduction | theory, xss, introduction, stored-xss, reflected-xss, dom-xss
xss/02-stored-xss.md | Section 2 — Stored XSS | xss, stored-xss, persistent-xss, javascript, cookie-stealing
xss/03-reflected-xss.md | Section 3 — Reflected XSS | xss, reflected-xss, non-persistent, get-parameter, url-payload
xss/04-dom-xss.md | Section 4 — DOM XSS | xss, dom-xss, innerhtml, client-side, source-sink, onerror
xss/05-xss-discovery.md | Section 5 — XSS Discovery | xss, reflected-xss, xsstrike, discovery, recon, web-attacks
xss/06-defacing.md | Section 6 — Defacing | xss, defacing, stored-xss, javascript, html
xss/07-xss-phishing.md | Section 7 — XSS Phishing Attack | xss, phishing, reflected-xss, credential-stealing, document.write, social-engineering
xss/09-xss-prevention.md | Section 9 — XSS Prevention | xss, prevention, defense, sanitization, input-validation, output-encoding
