# Section 9 — Tomcat - Discovery & Enumeration

## ID
900

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 9 — Tomcat - Discovery & Enumeration

## Description
Covers fingerprinting Apache Tomcat instances, understanding Tomcat directory structure and key config files, and enumerating the manager/host-manager admin pages for initial foothold opportunities.

## Tags
tomcat, enumeration, java, web-server, fingerprinting, gobuster

## Commands
- `curl -s http://<TARGET>:<PORT>/docs/ | grep Tomcat`
- `gobuster dir -u http://<TARGET>:<PORT>/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-small.txt`
- `curl -s http://<TARGET>:<PORT>/invalid`

## What This Section Covers
Apache Tomcat is a Java-based open-source web server commonly found on internal networks (and occasionally externally). This section walks through how to fingerprint a Tomcat instance, understand its directory layout and key configuration files (`tomcat-users.xml`, `web.xml`), and enumerate admin endpoints (`/manager`, `/host-manager`) that can lead to remote code execution via WAR file upload.

## Methodology
1. Identify Tomcat via the `Server` header — request an invalid page (e.g. `/invalid`) and check the HTTP 404 error page or response headers for the Tomcat version.
2. If custom error pages hide the version, hit `/docs/` and grep for the Tomcat version string with `curl -s http://<TARGET>:<PORT>/docs/ | grep Tomcat`.
3. Review the Tomcat directory structure — key files are `conf/tomcat-users.xml` (credentials + roles) and `webapps/*/WEB-INF/web.xml` (deployment descriptor with routes and servlet mappings).
4. Enumerate for `/manager` and `/host-manager` pages using `gobuster` or by browsing directly.
5. Attempt login with common weak/default credentials (`tomcat:tomcat`, `admin:admin`).
6. If login succeeds, WAR file upload gives RCE on the Tomcat server.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| What version of Tomcat is running on the application located at http://web01.inlanefreight.local:8180? | 10.0.10 | Add `STMIP web01.inlanefreight.local` to `/etc/hosts`, browse to `http://web01.inlanefreight.local:8180`, click "Documentation" — version shown in the docs page title |
| What role does the admin user have in the configuration example? | admin-gui | From the `tomcat-users.xml` example in the reading material — the admin user is assigned `manager-gui,admin-gui` roles, the answer is `admin-gui` |

## Key Takeaways
- Tomcat is far more common on internal pentests than external — EyeWitness flags it as a "High Value Target" and weak/default creds are frequent.
- Requesting an invalid page is the quickest fingerprinting trick — the default 404 page leaks the exact Tomcat version.
- `tomcat-users.xml` defines four built-in manager roles: `manager-gui`, `manager-script`, `manager-jmx`, `manager-status` — `manager-gui` is the one that gives HTML GUI access.
- `WEB-INF/web.xml` (the deployment descriptor) is a high-value file to target via LFI — it maps URL routes to Java servlet classes and can reveal business logic paths.
- The `webapps/` folder is the default webroot; each app inside it follows a standard structure with `WEB-INF/classes/` holding compiled Java classes that may contain sensitive logic.

## Gotchas
- Don't forget to add the vHost entries to `/etc/hosts` before attempting to reach the target (`app-dev.inlanefreight.local` and `web01.inlanefreight.local`).
- The admin user's role question asks specifically about the configuration *example* in the reading — the answer is `admin-gui`, not the full role string `manager-gui,admin-gui`.
