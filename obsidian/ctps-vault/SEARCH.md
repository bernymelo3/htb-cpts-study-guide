# CPTS Vault — Search Index

**Purpose:** flat keyword index for fast lookup. Grep this file FIRST, then Read the matching note.
**Format:** `module/file.md | Title | tags`
**Updated:** 2026-05-16 (regenerated)

---

AD-PRESENTATION-SLIDES.md | <untagged> | -
AD-PRESENTATION.md | <untagged> | -
AD-SIMPLE-GUIDE.md | <untagged> | -
ATTACK-PATHS.md | <untagged> | -
EXAM-WARNINGS.md | <untagged> | -
INDEX.md | <untagged> | -
METHODOLOGY-REGISTRY.md | <untagged> | -
_templates/methodology-prompt.md | <untagged> | -
ad-enum-attacks/00-METHODOLOGY.md | Active Directory Enumeration & Attacks — Full Pentest Methodology | methodology, active-directory, ad, exam, cheatsheet, decision-tree, kerberoasting, dcsync, acl-abuse, password-spraying, bloodhound, llmnr, petitpotam
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
attacking-common-applications/00-METHODOLOGY.md | Attacking Common Applications — Full Exploitation Methodology | methodology, web-applications, cms, app-servers, rce, exploit, default-creds, web-shell, cve, tomcat, jenkins, splunk, gitlab, coldfusion, joomla, wordpress, drupal, exam, cheatsheet
attacking-common-applications/09-tomcat-discovery-enumeration.md | Section 9 — Tomcat - Discovery & Enumeration | tomcat, enumeration, java, web-server, fingerprinting, gobuster
attacking-common-applications/1-intro.md | Section 1 — Introduction to Attacking Common Applications | theory, applications, web, methodology, introduction, enumeration
attacking-common-applications/10-attacking-tomcat.md | Section 10 — Attacking Tomcat | tomcat, brute-force, war-upload, rce, ghostcat, metasploit
attacking-common-applications/11-jenkins-discovery-enumeration.md | Section 11 — Jenkins - Discovery & Enumeration | jenkins, enumeration, fingerprinting, java, ci-cd, default-creds
attacking-common-applications/12-attacking-jenkins.md | Section 12 — Attacking Jenkins | jenkins, groovy, rce, script-console, reverse-shell, metasploit
attacking-common-applications/13-splunk-discovery-enumeration.md | Section 13 — Splunk - Discovery & Enumeration | splunk, enumeration, fingerprinting, siem, default-creds, scripted-inputs
attacking-common-applications/14-attacking-splunk.md | Section 14 — Attacking Splunk | splunk, rce, reverse-shell, custom-app, powershell, deployment-server
attacking-common-applications/15-prtg-network-monitor.md | Section 15 — PRTG Network Monitor | prtg, command-injection, cve-2018-9276, monitoring, powershell, rce
attacking-common-applications/16-osticket.md | Section 16 — osTicket | osticket, helpdesk, ticketing, credential-harvesting, information-disclosure
attacking-common-applications/17-gitlab-discovery-enumeration.md | Section 17 — GitLab - Discovery & Enumeration | gitlab, enumeration, git, credential-harvesting, user-enumeration, devops
attacking-common-applications/18-attacking-gitlab.md | Section 18 — Attacking GitLab | gitlab, user-enumeration, rce, exiftool, cve-2021-22205, reverse-shell
attacking-common-applications/19-attacking-tomcat-cgi.md | Section 19 — Attacking Tomcat CGI | tomcat, cgi, cve-2019-0232, command-injection, windows, url-encoding
attacking-common-applications/2-attacking_common_applications_app_discovery_enumeration.md | Section 2 — Application Discovery & Enumeration | enumeration, eyewitness, aquatone, nmap, web-discovery, recon
attacking-common-applications/20-attacking-cgi-shellshock.md | Section 20 — Attacking CGI Applications - Shellshock | shellshock, cgi, cve-2014-6271, reverse-shell, bash, gobuster
attacking-common-applications/21-attacking-thick-client-applications.md | Section 21 — Attacking Thick Client Applications | thick-client, reverse-engineering, hardcoded-credentials, procmon, dnspy, x64dbg, de4dot
attacking-common-applications/22-exploiting-web-vulns-thick-client.md | Section 22 — Exploiting Web Vulnerabilities in Thick-Client Applications | thick-client, sqli, path-traversal, java, jar, jd-gui, decompile
attacking-common-applications/23-coldfusion-discovery-enumeration.md | Section 23 — ColdFusion - Discovery & Enumeration | coldfusion, enumeration, discovery, nmap, cfml, adobe
attacking-common-applications/24-attacking-coldfusion.md | Section 24 — Attacking ColdFusion | coldfusion, rce, directory-traversal, searchsploit, fckeditor, cve-2009-2265, cve-2010-2861
attacking-common-applications/25-iis-tilde-enumeration.md | Section 25 — IIS Tilde Enumeration | iis, tilde, 8.3-shortname, gobuster, enumeration, windows
attacking-common-applications/26-attacking-ldap.md | Section 26 — Attacking LDAP | ldap, injection, authentication-bypass, wildcard, openldap, nmap
attacking-common-applications/28-attacking-apps-connecting-to-services.md | Section 28 — Attacking Applications Connecting to Services | gdb, peda, reverse-engineering, credentials, odbc, dnspy, dll, dotnet
attacking-common-applications/29-other-notable-applications.md | Section 29 — Other Notable Applications | weblogic, vcenter, zabbix, nagios, enumeration, metasploit, methodology
attacking-common-applications/3-attacking_common_applications_wordpress_discovery_enumeration.md | Section 3 — WordPress - Discovery & Enumeration | wordpress, wpscan, enumeration, cms, plugins, web
attacking-common-applications/30-application-hardening.md | Section 30 — Application Hardening | hardening, defense, best-practices, access-controls, authentication, theory
attacking-common-applications/31-skills-assessment-I.md | Skills Assessment I — Tomcat CGI Servlet RCE via CVE-2019-0232 | tomcat, cve-2019-0232, cgi, metasploit, meterpreter, gobuster
attacking-common-applications/32-skills-assessment-II.md | Skills Assessment II — vHost Enumeration → GitLab Credential Leak → Nagios XI Authenticated RCE | vhost, gobuster, wordpress, gitlab, nagios-xi, rce, credential-leak
attacking-common-applications/33-skills-assessment-III.md | Skills Assessment III — .NET DLL Reverse Engineering to Extract Hardcoded MSSQL Credentials | thick-client, dll, dnspy, reverse-engineering, mssql, hardcoded-credentials
attacking-common-applications/4-attacking_common_applications_attacking_wordpress.md | Section 4 — Attacking WordPress | wordpress, wpscan, brute-force, rce, lfi, xmlrpc, metasploit
attacking-common-applications/4-attacking_common_applications_joomla_discovery_enumeration.md | Section 5 — Joomla - Discovery & Enumeration | joomla, cms, enumeration, droopescan, brute-force, fingerprinting
attacking-common-applications/5attacking_common_applications_joomla_discovery_enumeration.md | Section 5 — Joomla - Discovery & Enumeration | joomla, cms, enumeration, droopescan, brute-force, fingerprinting
attacking-common-applications/6-attacking_common_applications_attacking_joomla.md | Section 6 — Attacking Joomla | joomla, rce, template-editor, directory-traversal, cve-2019-10945, web-shell
attacking-common-applications/7-attacking_common_applications_drupal_discovery_enumeration.md | Section 7 — Drupal - Discovery & Enumeration | drupal, cms, enumeration, droopescan, fingerprinting, nodes
attacking-common-applications/8-attacking_common_applications_attacking_drupal.md | Section 8 — Attacking Drupal | drupal, rce, drupalgeddon, php-filter, module-upload, cve
command-injetions/00-METHODOLOGY.md | Command Injections — Detect → Inject → Filter-Bypass → RCE Playbook | methodology, command-injection, exam, cheatsheet, decision-tree, injection-operators, filter-bypass, waf-bypass, ifs, base64, obfuscation, bashfuscator, host-checker, rce
command-injetions/01-intro.md | Section 1 — Intro to Command Injections | theory, concept, command-injection, injection-types, owasp
command-injetions/02-detection.md | Section 2 — Detection | command-injection, detection, injection-operators, ping, host-checker
command-injetions/03-injecting-commands.md | Section 3 — Injecting Commands | command-injection, burp-suite, front-end-bypass, url-encoding, input-validation
command-injetions/04-other-injection-operators.md | Section 4 — Other Injection Operators | command-injection, operators, and, or, pipe, burp-suite
command-injetions/05-command_injections_identifying_filters.md | Section 5 — Identifying Filters | command-injection, filter-evasion, blacklist, waf, burp-suite, operators
command-injetions/06-command_injections_bypassing_space_filters.md | Section 6 — Bypassing Space Filters | command-injection, filter-evasion, space-bypass, ifs, brace-expansion, tabs
command-injetions/07-command_injections_bypassing_other_blacklisted_chars.md | Section 7 — Bypassing Other Blacklisted Characters | command-injection, filter-evasion, character-bypass, env-variables, character-shifting
command-injetions/08-command_injections_bypassing_blacklisted_commands.md | Section 8 — Bypassing Blacklisted Commands | command-injection, filter-evasion, command-obfuscation, quotes, blacklist-bypass
command-injetions/09-command_injections_advanced_obfuscation.md | Section 9 — Advanced Command Obfuscation | command-injection, obfuscation, base64, case-manipulation, reverse-commands, waf-bypass
command-injetions/10-evasion-tools.md | Section 10 — Evasion Tools | evasion, obfuscation, bashfuscator, dosfuscation, command-injection, tools
command-injetions/11-defense.md | Section 11 — Command Injection Prevention | defence, prevention, secure-coding, input-validation, hardening
command-injetions/12-command_injections_skills_assessment.md | Skills Assessment — Web File Manager Command Injection | command-injection, filter-bypass, burp-suite, whitespace-bypass, slash-bypass, web-app
common-services/00-METHODOLOGY.md | Common Services Exploitation — End-to-End Playbook | methodology, smb, ftp, rdp, dns, email, initial-access, lateral-movement, enumeration, misconfigurations, brute-force, pass-the-hash, zone-transfer, exam, cheatsheet, decision-tree, null-session, anonymous
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
ffuf/00-overview.md | <untagged> | -
ffuf/01-introduction.md | Section 1 — Introduction | ffuf, fuzzing, intro, web, theory
ffuf/02-web-fuzzing.md | Section 2 — Web Fuzzing | ffuf, theory, seclists, wordlists, web-fuzzing
ffuf/03-directory-fuzzing.md | Section 3 — Directory Fuzzing | ffuf, directory-fuzzing, seclists, lab, web-content
ffuf/04-page-fuzzing.md | Section 4 — Page Fuzzing | ffuf, page-fuzzing, extension-fuzzing, lab, php
ffuf/05-recursive-fuzzing.md | Section 5 — Recursive Fuzzing | ffuf, recursive-fuzzing, lab, directory-tree
ffuf/06-dns-records.md | Section 6 — DNS Records | ffuf, theory, dns, etc-hosts, hostname-resolution
ffuf/07-sub-domain-fuzzing.md | Section 7 — Sub-domain Fuzzing | ffuf, sub-domain, dns, lab, public-dns
ffuf/08-vhost-fuzzing.md | Section 8 — Vhost Fuzzing | ffuf, vhost, host-header, lab, theory
ffuf/09-filtering-results.md | Section 9 — Filtering Results | ffuf, filtering, vhost, lab, calibration
ffuf/10-parameter-fuzzing-get.md | Section 10 — Parameter Fuzzing — GET | ffuf, parameter-fuzzing, get, query-string, lab
ffuf/11-parameter-fuzzing-post.md | Section 11 — Parameter Fuzzing — POST | ffuf, parameter-fuzzing, post, body, content-type
ffuf/12-value-fuzzing.md | Section 12 — Value Fuzzing | ffuf, value-fuzzing, custom-wordlist, lab, ids
ffuf/13-skills-assessment.md | Section 13 — Skills Assessment — Web Fuzzing | ffuf, skills-assessment, lab, end-to-end, vhost, recursion, parameters
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
file-transfers/00-overview.md | <untagged> | -
file-transfers/01-introduction.md | Section 1 — Introduction | file-transfer, intro, methodology, opsec, network-defenses
file-transfers/02-windows-file-transfer-methods.md | Section 2 — Windows File Transfer Methods | windows, powershell, smb, ftp, impacket, base64, webclient, invoke-webrequest
file-transfers/03-linux-file-transfer-methods.md | Section 3 — Linux File Transfer Methods | linux, wget, curl, bash, dev-tcp, scp, ssh, base64, python-http-server, uploadserver
file-transfers/04-transferring-files-with-code.md | Section 4 — Transferring Files with Code | python, php, ruby, perl, javascript, vbscript, cscript, one-liner, fileless
file-transfers/05-miscellaneous-file-transfer-methods.md | Section 5 — Miscellaneous File Transfer Methods | netcat, ncat, dev-tcp, powershell-remoting, winrm, ps-session, rdp, xfreerdp, rdesktop, mstsc, tsclient
file-transfers/06-protected-file-transfers.md | Section 6 — Protected File Transfers | encryption, aes, openssl, pbkdf2, exfil, opsec, invoke-aesencryption, data-protection
file-transfers/07-catching-files-over-https.md | Section 7 — Catching Files over HTTP/S | nginx, http-put, upload, webdav, dav-methods, catching-files, apache-vs-nginx
file-transfers/08-living-off-the-land.md | Section 8 — Living off The Land | lolbas, gtfobins, lolbins, certreq, certutil, bitsadmin, openssl, applocker, whitelisting
file-transfers/09-detection.md | Section 9 — Detection | detection, user-agent, defender, blue-team, fingerprinting, threat-hunting, ioc
file-transfers/10-evading-detection.md | Section 10 — Evading Detection | evasion, user-agent, lolbins, gfxdownloadwrapper, applocker, opsec, evasive-testing
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
footprinting/00-overview.md | <untagged> | -
footprinting/01-enumeration-principles.md | Section 1 — Enumeration Principles | enumeration, principles, mindset, methodology, theory
footprinting/02-enumeration-methodology.md | Section 2 — Enumeration Methodology | methodology, layers, framework, recon, pentest
footprinting/03-domain-information.md | Section 3 — Domain Information | domain, dns, crt.sh, certificate-transparency, shodan, dig, txt-records, passive-recon
footprinting/04-cloud-resources.md | Section 4 — Cloud Resources | cloud, aws, s3, azure, blob, gcp, google-dorks, grayhatwarfare, ssh-keys
footprinting/05-staff.md | Section 5 — Staff | osint, linkedin, github, job-postings, tech-stack, social-engineering, staff
footprinting/06-ftp.md | Section 6 — FTP | ftp, vsftpd, tftp, anonymous, ssl, openssl, nc, nmap, ftp-anon, ftp-syst
footprinting/07-smb.md | Section 7 — SMB | smb, samba, cifs, rpcclient, smbclient, smbmap, crackmapexec, enum4linux, null-session, rid-brute
footprinting/08-nfs.md | Section 8 — NFS | nfs, rpcbind, mount, showmount, nfs-ls, uid-gid, root-squash, sun-rpc
footprinting/09-dns.md | Section 9 — DNS | dns, bind, dig, axfr, zone-transfer, subdomain-bruteforce, dnsenum, named.conf
footprinting/10-smtp.md | Section 10 — SMTP | smtp, esmtp, postfix, vrfy, mail-spoofing, open-relay, mta, mua, msa, telnet
footprinting/11-imap-pop3.md | Section 11 — IMAP / POP3 | imap, pop3, dovecot, ssl, tls, curl, openssl, mailbox, mail-clients
footprinting/12-snmp.md | Section 12 — SNMP | snmp, snmpv2c, snmpv3, mib, oid, snmpwalk, onesixtyone, braa, community-string
footprinting/13-mysql.md | Section 13 — MySQL | mysql, mariadb, mssql, port-3306, nmap, sql, lamp, lemp
footprinting/14-mssql.md | Section 14 — MSSQL | mssql, microsoft-sql-server, port-1433, ssms, impacket, mssqlclient, metasploit, windows-auth, t-sql
footprinting/15-oracle-tns.md | Section 15 — Oracle TNS | oracle, tns, port-1521, odat, sqlplus, sid, sysdba, sys.user$, file-upload
footprinting/16-ipmi.md | Section 16 — IPMI | ipmi, bmc, hp-ilo, dell-idrac, supermicro, rakp, hashcat-7300, default-credentials, oob-management
footprinting/17-linux-remote-mgmt.md | Section 17 — Linux Remote Management Protocols | ssh, rsync, r-services, rlogin, rsh, rexec, rcp, rwho, rusers, openssh, ssh-audit, hosts.equiv, rhosts
footprinting/19-lab-easy.md | Section 19 — Footprinting Lab · Easy | htb, lab, footprinting, dns, axfr, dnsenum, proftpd, ftp, ssh, id_rsa, ceil, easy
footprinting/20-lab-medium.md | Section 20 — Footprinting Lab · Medium | htb, lab, footprinting, nfs, smb, rdp, mssql, ssms, password-reuse, windows, medium
footprinting/21-lab-hard.md | Section 21 — Footprinting Lab · Hard | htb, lab, footprinting, snmp, onesixtyone, snmpwalk, imap, ssh, id_rsa, mysql, hard, password-reuse
getting-started/00-METHODOLOGY.md | <untagged> | -
getting-started/01-infosec-fundamentals.md | <untagged> | -
getting-started/02-pentest-distro-setup.md | <untagged> | -
getting-started/03-service-scanning.md | <untagged> | -
getting-started/04-web-enumeration.md | <untagged> | -
getting-started/05-shell-types-setup.md | <untagged> | -
getting-started/06-privilege-escalation.md | <untagged> | -
getting-started/07-common-pitfalls.md | <untagged> | -
linux-privallege-escalation/00-METHODOLOGY.md | Linux Privilege Escalation — Full Exam Playbook (low-priv shell → root) | methodology, linux-privesc, lpe, exam, cheatsheet, decision-tree, stuck, got-shell, low-priv, suid, sgid, sudo -l, nopasswd, cron, path-abuse, wildcard, restricted-shell, rbash, capabilities, cap_setuid, lxd, docker, docker.sock, kubernetes, kubelet, logrotate, nfs, no_root_squash, tmux, kernel-exploit, overlayfs, ld_preload, shared-object, python-hijack, pythonpath, CVE-2019-14287, CVE-2021-3156, baron-samedit, CVE-2021-4034, pwnkit, polkit, pkexec, CVE-2022-0847, dirty-pipe, netfilter, gtfobins, pspy, root
linux-privallege-escalation/01-intro.md | Section 1 — Introduction to Linux Privilege Escalation | linux, privilege-escalation, enumeration, theory, methodology
linux-privallege-escalation/02_environment_enumeration.md | Section 2 — Environment Enumeration | linux-privesc, enumeration, environment, kernel, users, network
linux-privallege-escalation/03_services_internals_enumeration.md | Section 3 — Linux Services & Internals Enumeration | linux-privesc, services, cron, processes, config-files, internals
linux-privallege-escalation/04_credential_hunting.md | Section 4 — Credential Hunting | linux-privesc, credential-hunting, ssh-keys, web-config, passwords
linux-privallege-escalation/05_path_abuse.md | Section 5 — Path Abuse | linux-privesc, path-abuse, environment, hijacking
linux-privallege-escalation/06-wildcard-abuse.md | Section 6 — Wildcard Abuse | linux, privilege-escalation, wildcard, cron, tar, theory
linux-privallege-escalation/07-scaping-restricted-shells.md | Section 7 — Escaping Restricted Shells | restricted-shell, rbash, privilege-escalation, shell-escape, ssh-bypass, post-exploitation
linux-privallege-escalation/08-special-permissions.md | Section 8 — Special Permissions | suid, sgid, gtfobins, privilege-escalation, find, special-permissions
linux-privallege-escalation/09-sudo-rights-abuse.md | Section 9 — Sudo Rights Abuse | sudo, nopasswd, privilege-escalation, tcpdump, openssl, gtfobins
linux-privallege-escalation/10-privileged-groups.md | Section 10 — Privileged Groups | lxd, docker, disk, adm, privileged-groups, privilege-escalation
linux-privallege-escalation/11-capabilities.md | Section 11 — Capabilities | capabilities, cap_dac_override, getcap, setcap, vim, privilege-escalation
linux-privallege-escalation/12-vulnerable-services.md | Section 12 — Vulnerable Services | privesc, screen, suid, ld.so.preload, vulnerable-services, linux
linux-privallege-escalation/13-cron-job-abuse.md | Section 13 — Cron Job Abuse | privesc, cron, cronjob, reverse-shell, misconfiguration, linux
linux-privallege-escalation/14-lxd.md | Section 14 — LXD | privesc, lxd, lxc, containers, group-abuse, linux
linux-privallege-escalation/15-docker.md | Section 15 — Docker | privesc, docker, containers, group-abuse, docker-socket, linux
linux-privallege-escalation/16-kubernetes.md | Section 16 — Kubernetes | privesc, kubernetes, k8s, kubelet, containers, pod-escape
linux-privallege-escalation/17-lograte.md | Section 17 — Logrotate | privesc, logrotate, logrotten, race-condition, cron, linux
linux-privallege-escalation/18-miscellaneous-techniques.md | Section 18 — Miscellaneous Techniques | privesc, nfs, tmux, traffic-capture, no-root-squash, linux
linux-privallege-escalation/19-kernel-exploits.md | Section 19 — Kernel Exploits | kernel, cve, privesc, gcc, overlayfs, uname
linux-privallege-escalation/20-shared-libraries-ld-preload.md | Section 20 — Shared Libraries (LD_PRELOAD) | ld_preload, shared-libraries, sudo, privesc, gcc, env_keep
linux-privallege-escalation/21-shared-object-hijacking.md | Section 21 — Shared Object Hijacking | shared-object, suid, runpath, ldd, readelf, privesc
linux-privallege-escalation/22-python-library-hijacking.md | Section 22 — Python Library Hijacking | python, library-hijacking, pythonpath, suid, sudo, privesc
linux-privallege-escalation/23-sudo.md | Section 23 — Sudo | sudo, privilege-escalation, cve-2021-3156, cve-2019-14287, linux, 0-day
linux-privallege-escalation/24-polkit.md | Section 24 — Polkit | polkit, pkexec, pwnkit, cve-2021-4034, privilege-escalation, linux
linux-privallege-escalation/25-dirty-pipe.md | Section 25 — Dirty Pipe | dirty-pipe, cve-2022-0847, kernel, privilege-escalation, linux, pipes
linux-privallege-escalation/26-netfilter.md | Section 26 — Netfilter | netfilter, kernel, cve-2021-22555, cve-2022-25636, cve-2023-32233, privilege-escalation
linux-privallege-escalation/27-linux-hardening.md | Section 27 — Linux Hardening | hardening, defense, lynis, audit, configuration, linux
linux-privallege-escalation/28-skills-assessment.md | Skills Assessment — Linux Local Privilege Escalation | skills-assessment, privilege-escalation, tomcat, sudo, gtfobins, credential-hunting
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
nmap/00-overview.md | <untagged> | -
nmap/01-enumeration.md | Section 1 — Enumeration | enumeration, theory, methodology, mindset, recon
nmap/02-introduction-to-nmap.md | Section 2 — Introduction to Nmap | nmap, scanning, syn-scan, tcp-scan, udp-scan, reference, theory
nmap/03-host-discovery.md | Section 3 — Host Discovery | nmap, host-discovery, ping-sweep, arp, icmp, ttl
nmap/04-host-and-port-scanning.md | Section 4 — Host and Port Scanning | nmap, port-scanning, syn-scan, connect-scan, udp-scan, firewall, reference
nmap/05-saving-the-results.md | Section 5 — Saving the Results | nmap, output, reporting, xml, xsltproc, documentation
nmap/06-service-enumeration.md | Section 6 — Service Enumeration | nmap, service-detection, version-detection, banner-grabbing, netcat, tcpdump
nmap/07-nmap-scripting-engine.md | Section 7 — Nmap Scripting Engine (NSE) | nmap, nse, scripting, vuln, discovery, http-enum, banner
nmap/08-performance.md | Section 8 — Performance Tuning | nmap, performance, timing-templates, rtt, max-retries, min-rate
nmap/09-firewall-ids-ips-evasion.md | Section 9 — Firewall and IDS/IPS Evasion | nmap, evasion, firewall, ids, ips, decoy, source-port, ack-scan
nmap/10-evasion-easy-lab.md | Section 10 — Firewall and IDS/IPS Evasion · Easy Lab | nmap, lab, evasion, ids, os-fingerprinting, htb
nmap/11-evasion-medium-lab.md | Section 11 — Firewall and IDS/IPS Evasion · Medium Lab | nmap, lab, evasion, dns, udp, nsid, nse, htb
nmap/12-evasion-hard-lab.md | Section 12 — Firewall and IDS/IPS Evasion · Hard Lab | nmap, lab, evasion, source-port, ibm-db2, ftp, banner, htb
password-attacks/00-METHODOLOGY.md | Password Attacks — Full Credential-Theft Methodology | methodology, password-attacks, exam, cheatsheet, decision-tree, hashcat, john, cracking, password-spraying, credential-hunting, sam, lsass, ntds, dcsync, pass-the-hash, pth, pass-the-ticket, ptt, pass-the-certificate, ptc, mimikatz, pypykatz, secretsdump, netexec, keytab, hashcat-modes
password-attacks/00-overview.md | <untagged> | -
password-attacks/02-introduction-to-password-cracking.md | Section 2 — Introduction to Password Cracking | password-cracking, hashing, salt, rainbow-tables, brute-force, dictionary-attack, rockyou, theory
password-attacks/03-introduction-to-john-the-ripper.md | Section 3 — Introduction to John The Ripper | john-the-ripper, jtr, single-crack, wordlist-mode, incremental, hashid, 2john, ssh2john
password-attacks/04-introduction-to-hashcat.md | Section 4 — Introduction to Hashcat | hashcat, gpu, dictionary-attack, mask-attack, rules, best64, hashid
password-attacks/05-writing-custom-wordlists-and-rules.md | Section 5 — Writing Custom Wordlists and Rules | custom-wordlist, rules, osint, cewl, hashcat-rules, password-mutation
password-attacks/06-cracking-protected-files.md | Section 6 — Cracking Protected Files | ssh2john, office2john, pdf2john, encrypted-files, ssh-key, password-protected-docs
password-attacks/07-cracking-protected-archives.md | Section 7 — Cracking Protected Archives | zip2john, bitlocker2john, dislocker, openssl, gzip, vhd, archive-cracking
password-attacks/08-network-services.md | Section 8 — Network Services | winrm, ssh, rdp, smb, netexec, crackmapexec, hydra, evil-winrm, xfreerdp, smbclient
password-attacks/09-spraying-stuffing-defaults.md | Section 9 — Spraying, Stuffing, and Defaults | password-spraying, credential-stuffing, default-credentials, kerbrute, netexec, creds-cheat-sheet
password-attacks/10-windows-authentication-process.md | Section 10 — Windows Authentication Process | windows, authentication, lsass, sam, ntds, winlogon, kerberos, ntlm, credential-manager, dpapi
password-attacks/11-attacking-sam-system-security.md | Section 11 — Attacking SAM, SYSTEM, and SECURITY | sam, system, security-hive, reg-save, secretsdump, dpapi, dcc2, lsa-secrets, netexec, hashcat, ntlm
password-attacks/12-attacking-lsass.md | Section 12 — Attacking LSASS | lsass, mimikatz, pypykatz, comsvcs, minidump, wdigest, kerberos, dpapi, ntlm
password-attacks/13-attacking-windows-credential-manager.md | Section 13 — Attacking Windows Credential Manager | credential-manager, dpapi, cmdkey, runas, mimikatz, lazagne, credman, vaults
password-attacks/14-attacking-active-directory-and-ntds.md | Section 14 — Attacking Active Directory and NTDS.dit | active-directory, ntds, vssadmin, ntdsutil, kerbrute, username-anarchy, netexec, secretsdump, dcsync, pth, evil-winrm
password-attacks/15-credential-hunting-in-windows.md | Section 15 — Credential Hunting in Windows | credential-hunting, findstr, lazagne, windows-search, sysvol, unattend, web-config, keepass, gpp
password-attacks/16-linux-authentication-process.md | Section 16 — Linux Authentication Process | linux, pam, passwd, shadow, unshadow, sha512crypt, yescrypt, opasswd, hashcat-1800, jtr
password-attacks/17-credential-hunting-in-linux.md | Section 17 — Credential Hunting in Linux | credential-hunting, linux, mimipenguin, lazagne, firefox-decrypt, bash-history, configs, cronjobs, logs, browser
password-attacks/18-credential-hunting-in-network-traffic.md | Section 18 — Credential Hunting in Network Traffic | wireshark, pcredz, pcap, ftp, http, snmp, smtp, ntlmv2, kerberos, credit-cards, plaintext-protocols
password-attacks/19-credential-hunting-in-network-shares.md | Section 19 — Credential Hunting in Network Shares | smb, shares, snaffler, powerhuntshares, manspider, netexec-spider, smbclient, credential-hunting
password-attacks/20-pass-the-hash.md | Section 20 — Pass the Hash (PtH) | pth, pass-the-hash, ntlm, mimikatz, impacket, evil-winrm, netexec, invoke-thehash, xfreerdp, restricted-admin
password-attacks/21-pass-the-ticket-from-windows.md | Section 21 — Pass the Ticket (PtT) from Windows | ptt, pass-the-ticket, kerberos, mimikatz, rubeus, overpass-the-hash, kirbi, tgt, tgs, ccache, powershell-remoting
password-attacks/22-pass-the-ticket-from-linux.md | Section 22 — Pass the Ticket (PtT) from Linux | ptt, linux, kerberos, keytab, ccache, kinit, klist, sssd, realm, linikatz, keytabextract, krb5ccname
password-attacks/23-pass-the-certificate.md | Section 23 — Pass the Certificate | ptc, pkinit, adcs, esc8, shadow-credentials, ntlm-relay, pywhisker, pkinittools, gettgtpkinit, printerbug, dcsync
password-attacks/24-password-policies.md | Section 24 — Password Policies | password-policy, nist, cis, pci-dss, blacklist, defense, blue-team
password-attacks/25-password-managers.md | Section 25 — Password Managers | password-manager, bitwarden, 1password, keepass, fido2, mfa, totp, passwordless, defense
password-attacks/26-skills-assessment.md | Section 26 — Skills Assessment — Password Attacks | skills-assessment, password-attacks, credential-theft-shuffle, pivoting, ligolo-ng, snaffler, password-safe, mimikatz, ntds, dcsync
pivoting-tunneling/00-METHODOLOGY.md | Pivoting, Tunneling & Port Forwarding — Full Pentest Methodology | methodology, pivoting, tunneling, port-forwarding, socks, proxychains, ssh, meterpreter, socat, plink, sshuttle, rpivot, icmp-tunneling, ptunnel, dual-homed, dynamic-forward, local-forward, remote-forward, reverse-shell, multi-hop, internal-network, no-route, can-not-reach-internal, firewall-blocks-outbound, stuck, now-what, exam, cheatsheet, decision-tree
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
shells-payloads/00-overview.md | <untagged> | -
shells-payloads/01-shells-jack-us-in.md | Section 1 — Shells Jack Us In, Payloads Deliver Us Shells | shells, payloads, fundamentals, theory, cli, terminology
shells-payloads/02-cat5-engagement-prep.md | Section 2 — CAT5 Security's Engagement Preparation | overview, objectives, methodology, skills-assessment-prep
shells-payloads/03-anatomy-of-a-shell.md | Section 3 — Anatomy of a Shell | shell, terminal-emulator, bash, powershell, env, ps
shells-payloads/04-bind-shells.md | Section 4 — Bind Shells | bind-shell, netcat, mkfifo, fifo, pipes, listener, ports
shells-payloads/05-reverse-shells.md | Section 5 — Reverse Shells | reverse-shell, netcat, powershell, tcpclient, av-evasion, windows-defender
shells-payloads/06-introduction-to-payloads.md | Section 6 — Introduction to Payloads | payloads, one-liners, mkfifo, tcpclient, invoke-expression, .net, fifo
shells-payloads/07-automating-payloads-metasploit.md | Section 7 — Automating Payloads & Delivery with Metasploit | metasploit, msfconsole, psexec, smb, meterpreter, exploit-modules
shells-payloads/08-crafting-payloads-msfvenom.md | Section 8 — Crafting Payloads with MSFvenom | msfvenom, payloads, staged, stageless, elf, exe, social-engineering
shells-payloads/09-infiltrating-windows.md | Section 9 — Infiltrating Windows | windows, eternalblue, ms17-010, ttl, fingerprinting, cmd, powershell, payload-types
shells-payloads/10-infiltrating-linux.md | Section 10 — Infiltrating Unix/Linux | linux, rconfig, php, msf-module-loading, tty-upgrade, python-pty, jail-shell
shells-payloads/11-spawning-interactive-shells.md | Section 11 — Spawning Interactive Shells | tty, jail-shell, shell-upgrade, perl, ruby, lua, awk, find, vim, sudo
shells-payloads/12-introduction-to-web-shells.md | Section 12 — Introduction to Web Shells | web-shells, file-upload, php, jsp, aspx, foothold, external-pentest
shells-payloads/13-laudanum-webshell.md | Section 13 — Laudanum, One Webshell to Rule Them All | laudanum, web-shells, aspx, asp, jsp, php, ip-allowlist, ascii-comments
shells-payloads/14-antak-webshell.md | Section 14 — Antak Webshell | antak, nishang, aspx, powershell, web-shells, ippsec, asp-net
shells-payloads/15-php-web-shells.md | Section 15 — PHP Web Shells | php, web-shells, burp, content-type-bypass, rconfig, whitewinterwolf, file-upload-bypass
shells-payloads/16-skills-assessment-live-engagement.md | Section 16 — The Live Engagement | skills-assessment, tomcat, war, eternalblue, ms17-010, msfvenom, lightweight-fb-blog, foothold, pivot
shells-payloads/17-detection-and-prevention.md | Section 17 — Detection & Prevention | detection, defense, blue-team, mitre-attack, c2, netflow, defender, mitigations
sql-injection-fundamentals/00-METHODOLOGY.md | SQL Injection Fundamentals — Full Exploitation Methodology | methodology, sql-injection, union-injection, authentication-bypass, database-enumeration, file-write, rce, exam, decision-tree, cheatsheet
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
using-the-metasploit/00-overview.md | <untagged> | -
using-the-metasploit/01-preface.md | Section 1 — Preface | metasploit, philosophy, methodology, discipline
using-the-metasploit/02-introduction-to-metasploit.md | Section 2 — Introduction to Metasploit | metasploit, msfconsole, architecture, msf-pro, filesystem-layout
using-the-metasploit/03-introduction-to-msfconsole.md | Section 3 — Introduction to MSFconsole | metasploit, msfconsole, update, engagement-structure, enumeration
using-the-metasploit/04-modules.md | Section 4 — Modules | metasploit, modules, search, ms17-010, eternalromance, exploitation
using-the-metasploit/05-targets.md | Section 5 — Targets | metasploit, targets, return-address, msfpescan, exploit-modules
using-the-metasploit/06-payloads.md | Section 6 — Payloads | metasploit, payloads, meterpreter, staged-payloads, reverse-tcp, bind-tcp, lhost, lport
using-the-metasploit/07-encoders.md | Section 7 — Encoders | metasploit, encoders, msfvenom, shikata-ga-nai, av-evasion, bad-characters, virustotal
using-the-metasploit/08-databases.md | Section 8 — Databases | metasploit, database, postgresql, msfdb, workspace, db_nmap, db_import, hosts, services, creds, loot
using-the-metasploit/09-plugins-and-mixins.md | Section 9 — Plugins & Mixins | metasploit, plugins, mixins, nessus, darkoperator, ruby
using-the-metasploit/10-sessions-and-jobs.md | Section 10 — Sessions & Jobs | metasploit, sessions, jobs, background, bg, post-exploitation, privilege-escalation, elfinder, sudo-baron-samedit, cve-2021-3156
using-the-metasploit/11-meterpreter.md | Section 11 — Meterpreter | metasploit, meterpreter, post-exploitation, token-impersonation, privilege-escalation, hashdump, lsa-secrets, fortilogger, ms15-051, dll-injection
using-the-metasploit/12-writing-importing-modules.md | Section 12 — Writing & Importing Modules | metasploit, modules, exploit-db, searchsploit, porting, ruby, mixins, reload-all, loadpath
using-the-metasploit/13-msfvenom.md | Section 13 — Introduction to MSFVenom | metasploit, msfvenom, payload-generation, aspx-shell, multi-handler, local-exploit-suggester, ms10-015-kitrap0d, ftp-upload, iis
using-the-metasploit/14-firewall-ids-ips-evasion.md | Section 14 — Firewall & IDS/IPS Evasion | metasploit, evasion, ids, ips, av-bypass, msfvenom, packers, executable-templates, archives, rar, virustotal, offset, nop-sled
using-the-metasploit/15-msf-updates.md | Section 15 — Metasploit Framework Updates (August 2020 / MSF6) | metasploit, msf6, msf5, aes-encryption, smbv3, polymorphic-shellcode, kiwi, mimikatz, changelog
web-attacks/01-attacks.md | Section 1 — Introduction to Web Attacks | theory, concept, web-attacks, http-verb-tampering, idor, xxe
web-attacks/02-http-verb-tampering.md | Section 2 — Intro to HTTP Verb Tampering | theory, concept, http-verb-tampering, authentication-bypass, web-attacks
web-attacks/03-bypassing-basic-authentication.md | Section 3 — Bypassing Basic Authentication | http-verb-tampering, authentication-bypass, head-method, apache, web-server-misconfiguration
web-attacks/04-bypassing-security-filters.md | Section 4 — Bypassing Security Filters | http-verb-tampering, command-injection, security-filter-bypass, insecure-coding, php
web-attacks/05-verb-tampering-prevention.md | Section 5 — Verb Tampering Prevention | defense, prevention, http-verb-tampering, secure-coding, server-hardening
web-attacks/06-idor.md | Section 6 — Intro to IDOR | theory, idor, access-control, web-attacks, vulnerability
web-attacks/07-idor-2.md | Section 7 — Identifying IDORs | idor, discovery, parameter-fuzzing, access-control, web-attacks
web-attacks/08-mass-idor-enumeration.md | Section 8 — Mass IDOR Enumeration | idor, mass-enumeration, bash-scripting, parameter-tampering, document-exfiltration
web-attacks/09-bypassing-encoded-references.md | Section 9 — Bypassing Encoded References | idor, encoded-references, base64, md5, mass-enumeration, bash-scripting
web-attacks/10-idor-insecure-apis.md | Section 10 — IDOR in Insecure APIs | idor, api, information-disclosure, rest-api, access-control, json
web-attacks/11-chaining-idor-vulnerabilities.md | Section 11 — Chaining IDOR Vulnerabilities | idor, chained-attack, api, privilege-escalation, uuid-leak, burp-suite
web-attacks/12-idor-prevention.md | Section 12 — IDOR Prevention | theory, idor, access-control, rbac, defensive, prevention
web-attacks/13-intro-to-xxe.md | Section 13 — Intro to XXE | theory, xxe, xml, injection, owasp-top-10, web-attacks
web-attacks/14-xxe-local-file-disclosure.md | Section 14 — Local File Disclosure (XXE) | xxe, xml, lfi, php-filter, burp-suite, file-disclosure
web-attacks/15-xxe-advanced-file-disclosure.md | Section 15 — Advanced File Disclosure (XXE) | xxe, cdata, parameter-entities, error-based, dtd, file-disclosure
web-attacks/16-xxe-blind-data-exfiltration.md | Section 16 — Blind Data Exfiltration (XXE) | xxe, oob, blind, exfiltration, php-filter, xxeinjector
web-attacks/18-web-attacks-skills-assessment.md | Skills Assessment — Web Attacks | idor, verb-tampering, xxe, php-filter, skills-assessment, chained-attack
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
web-recon/00-overview.md | <untagged> | -
web-recon/01-introduction.md | Section 1 — Introduction | web-recon, methodology, active-recon, passive-recon, osint, attack-surface
web-recon/02-whois.md | Section 2 — WHOIS | whois, passive-recon, registrar, registrant, name-server, iana
web-recon/03-utilizing-whois.md | Section 3 — Utilizing WHOIS | whois, lab, iana-id, admin-email, grep, phishing, c2, threat-intel
web-recon/04-dns.md | Section 4 — DNS | dns, theory, zone-file, record-types, hosts-file, resolver, recursive-lookup
web-recon/05-digging-dns.md | Section 5 — Digging DNS | dns, dig, lab, ptr, mx, reverse-dns, nslookup, host, dnsrecon, theharvester
web-recon/06-subdomains.md | Section 6 — Subdomains | subdomains, theory, enumeration, active-recon, passive-recon, ct-logs, dev-staging
web-recon/07-subdomain-bruteforcing.md | Section 7 — Subdomain Bruteforcing | subdomain, brute-force, dnsenum, fierce, dnsrecon, amass, assetfinder, puredns, seclists
web-recon/08-dns-zone-transfers.md | Section 8 — DNS Zone Transfers | zone-transfer, axfr, dns, dig, lab, misconfiguration, soa, secondary-server
web-recon/09-virtual-hosts.md | Section 9 — Virtual Hosts | vhost, virtual-host, host-header, gobuster, ffuf, feroxbuster, fuzzing, hosts-file
web-recon/10-certificate-transparency-logs.md | Section 10 — Certificate Transparency Logs | ct-logs, crt.sh, censys, passive-recon, subdomain-discovery, ssl, tls, san
web-recon/11-fingerprinting.md | Section 11 — Fingerprinting | fingerprinting, banner-grabbing, headers, wafw00f, nikto, wappalyzer, whatweb, cms, waf
web-recon/12-crawling.md | Section 12 — Crawling | crawling, spidering, breadth-first, depth-first, links, comments, metadata, sensitive-files
web-recon/13-robots-txt.md | Section 13 — robots.txt | robots-txt, crawling, recon, hidden-paths, disallow, allow, sitemap, honeypot
web-recon/14-well-known-uris.md | Section 14 — .Well-Known URIs | well-known, rfc-8615, openid-connect, security-txt, mta-sts, oidc, recon, metadata
web-recon/15-creepy-crawlies.md | Section 15 — Creepy Crawlies | crawling, scrapy, reconspider, burp-spider, zap, apache-nutch, results-json, jq
web-recon/16-search-engine-discovery.md | Section 16 — Search Engine Discovery | search-engine, google-dork, osint, passive-recon, site-operator, filetype, ghdb
web-recon/17-web-archives.md | Section 17 — Web Archives | wayback-machine, web-archive, internet-archive, passive-recon, historical-data, snapshot
web-recon/18-automating-recon.md | Section 18 — Automating Recon | automation, finalrecon, recon-ng, theharvester, spiderfoot, osint-framework, framework
web-recon/19-skills-assessment.md | Section 19 — Skills Assessment | skills-assessment, lab, capstone, whois, gobuster-vhost, robots-txt, reconspider, full-chain
windows-privesc/00-METHODOLOGY.md | Windows Privilege Escalation — Full Local-to-SYSTEM Methodology | methodology, windows, windows-privesc, privesc, exam, cheatsheet, decision-tree, seimpersonate, sedebugprivilege, setakeownership, sebackupprivilege, seloaddriver, potato, printspoofer, juicypotato, backup-operators, server-operators, dnsadmins, event-log-readers, print-operators, hyper-v, unquoted-service-path, weak-service-permissions, alwaysinstallelevated, uac-bypass, printnightmare, hivenightmare, kernel-exploit, ms16-032, citrix-breakout, unattend-xml, lazagne, mremoteng, credential-hunting, scf, responder
windows-privesc/01-intro.md | Section 1 — Introduction to Windows Privilege Escalation | theory, windows, privilege-escalation, methodology, introduction, concept
windows-privesc/02-intro.md | Section 2 — Useful Tools | theory, windows, privilege-escalation, tools, enumeration, methodology
windows-privesc/03-situational-awareness.md | Section 3 — Situational Awareness | situational-awareness, network-enumeration, applocker, windows-defender, arp, dual-homed
windows-privesc/04-initial-enumeration.md | Section 4 — Initial Enumeration | initial-enumeration, whoami, systeminfo, netstat, tasklist, windows-privesc
windows-privesc/05-communication-with-processes.md | Section 5 — Communication with Processes | named-pipes, netstat, accesschk, network-services, process-communication, windows-privesc
windows-privesc/07-seimpersonate.md | Section 7 — SeImpersonate and SeAssignPrimaryToken | seimpersonate, privilege-escalation, potato, printspoofer, mssql, windows
windows-privesc/08-sedebugprivilege.md | Section 8 — SeDebugPrivilege | sedebugprivilege, lsass, mimikatz, procdump, credential-dumping, windows
windows-privesc/09-setakeownershipprivilege.md | Section 9 — SeTakeOwnershipPrivilege | setakeownershipprivilege, file-ownership, icacls, takeown, privilege-escalation, windows
windows-privesc/10-backup-operators.md | Section 10 — Windows Built-in Groups (Backup Operators) | backup-operators, sebackupprivilege, ntds.dit, diskshadow, secretsdump, windows
windows-privesc/11-event-log-readers.md | Section 11 — Event Log Readers | event-log-readers, credential-hunting, wevtutil, get-winevent, windows
windows-privesc/12-dnsadmins.md | Section 12 — DnsAdmins | dnsadmins, dll-injection, dnscmd, privilege-escalation, domain-controller, windows
windows-privesc/13-hyperv-admin.md | Section 13 — Hyper-V Administrators | theory, windows, hyper-v, privilege-escalation, group-membership, credential-theft
windows-privesc/14-print-operators.md | Section 14 — Print Operators | print-operators, seloaddriverprivilege, capcom, driver-load, privilege-escalation, windows
windows-privesc/15-server-operators.md | Section 15 — Server Operators | server-operators, service-hijack, sc-config, privilege-escalation, domain-controller, windows
windows-privesc/16-user-account-control.md | Section 16 — User Account Control | uac, privilege-escalation, dll-hijacking, windows, msfvenom, bypass
windows-privesc/17-weak-permissions.md | Section 17 — Weak Permissions | weak-permissions, service-hijacking, dll-hijacking, acl, privilege-escalation, windows
windows-privesc/18-kernel-exploits.md | Section 18 — Kernel Exploits | kernel-exploit, privilege-escalation, windows, hivenightmare, printnightmare, cve-2020-0668, patching
windows-privesc/19-vulnerable-services.md | Section 19 — Vulnerable Services | vulnerable-services, druva-insync, rpc, command-injection, privilege-escalation, windows
windows-privesc/20-dll-injection.md | Section 13 — Hyper-V Administrators | theory, windows, hyper-v, group-membership, privilege-escalation, credential-theft
windows-privesc/21-credential-hunting-notes.md | Section 21 — Credential Hunting | credential-hunting, findstr, dpapi, powershell-history, windows-privesc
windows-privesc/22-other-files-notes.md | Section 22 — Other Files | credential-hunting, findstr, sticky-notes, pssqlite, file-search, windows-privesc
windows-privesc/23-further-credential-theft-notes.md | Section 23 — Further Credential Theft | lazagne, sharpchrome, sessiongopher, keepass, registry-creds, windows-privesc
windows-privesc/24-citrix-breakout.md | Section 24 — Citrix Breakout | citrix, breakout, dialog-box, uac-bypass, alwaysinstallelevated, privilege-escalation
windows-privesc/25-interacting-with-users.md | Section 25 — Interacting with Users | scf, responder, ntlmv2, hashcat, wireshark, credential-theft
windows-privesc/26-pillaging.md | Section 26 — Pillaging | pillaging, mremoteng, cookies, restic, secretsdump, credential-theft
windows-privesc/27-miscellaneous-techniques.md | Section 27 — Miscellaneous Techniques | lolbas, alwaysinstallelevated, cve-2019-1388, scheduled-tasks, vhdx, credential-hunting
windows-privesc/28 — legacy-operating-systems.md | Section 28 — Legacy Operating Systems | theory, windows, legacy, end-of-life, privilege-escalation, exploitation
windows-privesc/29-windows-server-privesc-notes.md | Section 29 — Windows Server (Legacy Privesc) | windows-server-2008, privesc, sherlock, metasploit, ms10-092, legacy-os
windows-privesc/30-windows-desktop-versions-privesc-notes.md | Section 30 — Windows Desktop Versions (Legacy Privesc) | windows-7, privesc, ms16-032, windows-exploit-suggester, legacy-os, secondary-logon
windows-privesc/31-windows-hardening .md | Section 31 — Windows Hardening | theory, windows, hardening, defense, best-practices, configuration-management
windows-privesc/32-winprivesc-skills-assessment-part1.md | Skills Assessment — Part I (PrintNightmare + Credential Theft) | printnightmare, lazagne, command-injection, privilege-escalation, credential-theft, rdp
windows-privesc/33-winprivesc-skills-assessment-part2.md | Skills Assessment — Part II (Unattend.xml + AlwaysInstallElevated) | unattend-xml, alwaysinstallelevated, msfvenom, pwdump, hashcat, credential-hunting
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
