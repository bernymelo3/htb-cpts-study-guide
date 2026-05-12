# Section 11 — Jenkins - Discovery & Enumeration

## ID
902

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 11 — Jenkins - Discovery & Enumeration

## Description
Covers fingerprinting Jenkins instances, understanding their default configuration and authentication options, and identifying weak or default credentials for initial access.

## Tags
jenkins, enumeration, fingerprinting, java, ci-cd, default-creds

## Commands
- `curl -s http://<TARGET>:8000/login`
- `nmap -sV -p 8000,5000 <TARGET>`

## What This Section Covers
Jenkins is a Java-based open-source CI/CD automation server that runs in servlet containers like Tomcat. It's extremely common in enterprise environments (86,000+ companies) and is a high-value target because it frequently runs as `SYSTEM` on Windows, giving an immediate AD foothold if compromised. This section covers how to discover and fingerprint Jenkins and attempt login with weak/default credentials.

## Methodology
1. Jenkins runs on Tomcat port 8080 by default (this lab uses port 8000). Port 5000 is used for master-slave communication — seeing either port is a discovery indicator.
2. Fingerprint Jenkins by its distinctive login page at `/login`. The version number is displayed in the bottom-right corner of the web UI.
3. Jenkins can authenticate via its built-in database, LDAP, Unix user database, servlet container delegation, or no authentication at all. The default installation uses Jenkins' own database and does not allow self-registration.
4. Attempt login with common weak/default credentials — `admin:admin` is the most common. On internal pentests, it's not unusual to find Jenkins instances with no authentication enabled at all.
5. Once logged in, browse the UI to confirm the version number and begin enumerating configured jobs, credentials, and nodes for further attack surface.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Log in to the Jenkins instance. Browse around and submit the version number. | 2.303.1 | Add `STMIP jenkins.inlanefreight.local` to `/etc/hosts`, browse to `http://jenkins.inlanefreight.local:8000`, login with `admin:admin`, version shown at bottom-right of the page |

## Key Takeaways
- Jenkins on Windows frequently runs as `SYSTEM` — RCE on Jenkins means domain foothold with the highest local privilege.
- The default install uses Jenkins' own credential store and blocks self-registration, but weak creds like `admin:admin` are extremely common, especially on internal networks.
- Jenkins version is always visible at the bottom-right of the UI once authenticated — useful for checking known CVEs.
- Port 5000 (master-slave comms) is a secondary fingerprinting indicator if port 8080/8000 alone isn't conclusive.
- Jenkins was originally called Hudson (2005), renamed in 2011 after an Oracle dispute — occasionally you'll still see "Hudson" references in older installs.
