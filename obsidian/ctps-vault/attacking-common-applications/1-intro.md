## ID
606

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 1 — Introduction to Attacking Common Applications

## Description
Introduces the landscape of web-based and common applications encountered during penetration tests, their prevalence, typical vulnerabilities and misconfigurations, and the mindset needed to abuse built-in functionality for foothold, lateral movement, and data access.

## Tags
theory, applications, web, methodology, introduction, enumeration

## TL;DR — What's Important
- **Applications are everywhere:** CMS, dev tools, monitoring, ticketing, and code repositories are ubiquitous in internal and external environments and frequently serve as the initial foothold.
- **Known CVEs and built-in abuse:** Attackers gain RCE not only through classic vulnerabilities (SQLi, file upload) but by abusing legitimate admin features like script consoles, Groovy scripts, or task runners.
- **Default credentials remain a top finding:** Even sophisticated tools like Nexus Repository, Jenkins, and Splunk are often left with `admin:admin` or similar default logins, enabling full compromise.
- **Same app, different environment:** An application may be secure in one client's network but misconfigured, outdated, or overly privileged (e.g., running as SYSTEM) in another. Always test.
- **Market share doesn't equal security:** Apps with huge install bases (WordPress ~70% of CMS market) attract attackers, but niche tools often have weaker security because they are less scrutinized.

## Concept Overview
This module focuses on attacking common web applications and on-premise software that penetration testers repeatedly encounter. These range from Content Management Systems (WordPress, Drupal, Joomla) and servlet containers (Tomcat, Jenkins) to SIEM (Splunk), network monitoring (PRTG), ticketing (osTicket), and development platforms (GitLab). The core message is that while applications may differ, the attack methodology remains consistent: enumerate version, check for default credentials, map accessible functionality, look for known CVEs, and—crucially—determine how built-in admin or user features can be turned into code execution or data theft.

## Key Concepts

### Applications as Attack Surface
- **External and internal exposure:** Web applications are often reachable from the internet due to remote work, cloud migration, or misconfigured firewalls. They provide a direct path into the internal network.
- **2021 survey data:** 72% of organizations suffered at least one breach due to an application vulnerability. Top challenges included bot attacks (43%), supply chain attacks (39%), vulnerability detection (38%), and API security (37%).
- **Variety of types:** CMS, application servers, SIEM, network management, IT management, search engines, software configuration management, development tools, and enterprise integration brokers. Each has its own admin panel, scripting engine, or credential storage that can be abused.

### Categories and Common Applications
| Category | Example Applications |
|----------|----------------------|
| Web Content Management | WordPress, Drupal, Joomla, DotNetNuke |
| Application Servers | Apache Tomcat, Phusion Passenger, Oracle WebLogic, IBM WebSphere |
| SIEM | Splunk, Trustwave, LogRhythm |
| Network Management | PRTG Network Monitor, ManageEngine OpManager |
| IT Management | Nagios, Puppet, Zabbix, ManageEngine ServiceDesk Plus |
| Software Frameworks | JBoss, Axis2 |
| Customer Service Mgmt | osTicket, Zendesk |
| Search Engines | Elasticsearch, Apache Solr |
| Configuration Mgmt | Atlassian JIRA, GitHub, GitLab, Bugzilla, Bitbucket |
| Development Tools | Jenkins, Atlassian Confluence, phpMyAdmin |
| Enterprise Integration | Oracle Fusion Middleware, BizTalk, Apache ActiveMQ |

### Why Applications Are Vulnerable
- **Default credentials:** Admin panels left with factory passwords (e.g., `admin:admin123` for Nexus, `tomcat:tomcat` for Tomcat).
- **Unpatched known flaws:** Public CVEs for RCE, SQL injection, and file read exist for virtually all common platforms; patch lag is widespread.
- **Misconfigurations:** Overly permissive file uploads, exposed API endpoints, disabled authentication (dev mode), weak encryption, and verbose error messages that leak versions.
- **Functional abuse:** Many applications provide script execution (Jenkins script console, ManageEngine script runner, Nexus Groovy tasks) as a feature. When administrative access is obtained, this feature becomes a code execution vector.

### Methodology for Application Assessments
1. **Discovery:** Use enumeration tools (Nmap, EyeWitness, Aquatone) and manual techniques to identify running applications and their versions.
2. **Vulnerability mapping:** Cross-reference version with Exploit-DB, CVE databases, and known default credentials.
3. **Authentication testing:** Try default credentials, common weak passwords, and bypass techniques.
4. **Functional analysis:** After login (even low-privileged), map every feature—file upload, script execution, task scheduling, API calls, webhooks—to identify RCE or data exfiltration paths.
5. **Exploitation:** Execute known exploit payloads or craft custom abuse using built-in functionality, always targeting minimal privilege escalation (e.g., from user to admin within the app) first.
6. **Post-exploitation:** Extract configuration files (database credentials, API keys), pivot to other services, or elevate to SYSTEM if the application runs with high privileges.

### Lab and Host Setup
- The module uses virtual hosts (Vhosts) such as `app.inlanefreight.local`, `dev.inlanefreight.local` to simulate a multi-application environment on a single target IP.
- You must edit `/etc/hosts` (or equivalent) to map the target IP to all Vhost FQDNs.
- Example command to append entries:  
  `printf "%s\t%s\n\n" "$IP" "app.inlanefreight.local dev.inlanefreight.local blog.inlanefreight.local" | sudo tee -a /etc/hosts`
- Always verify the hosts file after spawning a target; exercises will display the required FQDNs.

## Why It Matters
In both internal and external penetration tests, the web application layer is frequently the path of least resistance. A misconfigured WordPress plugin, a Tomcat manager interface with default credentials, or a Splunk instance with an exposed scripted input can bypass years of network hardening. Knowing how to assess these applications efficiently—not just looking for SQLi, but understanding what the app does by design—separates a junior assessor from a competent one. The skills acquired here translate directly to real-world findings and often lead to critical‑risk severity demonstrations.

## Defender Perspective
- **Mitigations:**
  - **Change default credentials immediately**. Implement a strong password policy and, where possible, MFA.
  - **Apply patches promptly** for all internet‑facing and internal applications. Subscribe to vendor security announcements.
  - **Segregate applications** from sensitive internal networks; run them on dedicated VLANs with strict firewall rules.
  - **Disable unnecessary features**: script consoles, Groovy tasks, and other admin‑only code execution features should be turned off or tightly monitored.
  - **Enforce least privilege**: run application services with low‑privileged accounts (not SYSTEM or root) and use application whitelisting to prevent unintended executables.
  - **Regularly audit** application configurations and perform authenticated vulnerability scans.
- **Detection:** Monitor web server logs for unusual access to admin panels, failed login bursts, or pattern‑based exploit attempts. Use WAF rules tuned for known CVEs. For authenticated abuse, log and alert on script console usage, task creation, and file uploads. Sysmon or equivalent can capture unexpected child processes spawned by the application.
- **MITRE ATT&CK:**  
  - TA0001 (Initial Access) – Exploit Public‑Facing Application (T1190).  
  - TA0002 (Execution) – Command and Scripting Interpreter (T1059) via built‑in scripts.  
  - TA0005 (Defense Evasion) – Valid Accounts (T1078) using default credentials.  
  - TA0010 (Exfiltration) – Automated Exfiltration (T1020) from data stored in applications.

## Key Takeaways
- Always treat an application’s feature set as a potential weapon. What looks like a convenient “Run Script” button is a backdoor when you have admin access.
- Don’t focus solely on public exploits. Understanding the underlying technology (PHP, Java, Ruby) helps you find logic flaws even in unknown apps.
- Default credentials are still the #1 quick win. Build a comprehensive list and test them religiously.
- The market‑share data (WordPress 70%) means you’ll encounter these apps constantly; deep knowledge of a handful of platforms pays off repeatedly.
- The story about Nexus Repository illustrates that unfamiliar applications can be broken down by systematically studying their docs, API, and features—a mindset that works everywhere.

## Gotchas
- Vhosts can be elusive: if a scan shows port 80 open but a browser gives a generic page, you may need to add FQDNs to your hosts file before the application loads correctly.
- Some applications block version disclosure; use subtle cues like error pages, file paths (`/CHANGELOG.txt`, `/readme.html`), or specific cookie names.
- Abusing built‑in scripts may trigger antivirus or EDR when the script spawns cmd.exe or connects to an external IP; consider indirect execution or living‑off‑the‑land payloads.
- In shared environments, be careful not to take down a production application with a destructive exploit. Test on a non‑production instance when possible or coordinate with the client.