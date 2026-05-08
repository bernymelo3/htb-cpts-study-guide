# NOTE — File Inclusion / File Inclusion Prevention

## ID
601

## Module
File Inclusion

## Kind
lab

## Title
Section 10 — File Inclusion Prevention

## Description
Harden a live Apache server against LFI by locating the correct php.ini, disabling dangerous PHP functions, and confirming the block via the Apache error log.

## Tags
lfi, prevention, php.ini, hardening, disable-functions, apache

## Commands
- ssh htb-student@<IP>
- sudo find / -name php.ini
- sudo nano /etc/php/7.4/apache2/php.ini
- sudo service apache2 restart
- sudo su -
- echo "<?php system('id'); ?>" > /var/www/html/shell.php
- sudo tail -f /var/log/apache2/error.log

## What This Section Covers
File inclusion vulnerabilities can be mitigated at multiple layers: sanitizing user input, using whitelists, locking down php.ini, and deploying a WAF. This lab focuses on the server-hardening layer — locating the correct php.ini for Apache, disabling dangerous PHP functions via `disable_functions`, and confirming the block is enforced by reading the Apache error log.

## Methodology
1. SSH into the target with `htb-student:HTB_@cademy_stdnt!`
2. Run `sudo find / -name php.ini` — two results appear; the Apache one is `/etc/php/7.4/apache2/php.ini`
3. Edit that file at line 312 and populate the `disable_functions` directive
4. Restart Apache with `sudo service apache2 restart`
5. Drop a test web shell calling `system()` into the webroot
6. Monitor `/var/log/apache2/error.log` with `tail -f` while triggering the shell via browser — the warning confirms the block

## Multi-step Workflow
```bash
# Step 1 — SSH in
ssh htb-student@<IP>

# Step 2 — locate both php.ini files
sudo find / -name php.ini
# /etc/php/7.4/cli/php.ini       ← CLI only, irrelevant
# /etc/php/7.4/apache2/php.ini   ← this one

# Step 3 — edit Apache's php.ini at line 312
sudo nano /etc/php/7.4/apache2/php.ini
# Set:
# disable_functions=exec,passthru,shell_exec,system,proc_open,popen,curl_exec,curl_multi_exec,parse_ini_file,show_source

# Step 4 — restart Apache to apply changes
sudo service apache2 restart

# Step 5 — create a test shell
sudo su -
echo "<?php system('id'); ?>" > /var/www/html/shell.php

# Step 6 — watch the error log, then visit http://<IP>/shell.php in browser
sudo tail -f /var/log/apache2/error.log
# Expected: PHP Warning: system() has been disabled for security reasons
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Full path to php.ini for Apache | `/etc/php/7.4/apache2/php.ini` | `sudo find / -name php.ini` — second result is the Apache one |
| system() has been disabled for \_\_\_\_ reasons | `security` | Error log output after triggering shell.php |

## Key Takeaways
- Two php.ini files exist: CLI (`/etc/php/7.4/cli/php.ini`) and Apache (`/etc/php/7.4/apache2/php.ini`) — always edit the Apache one for web hardening
- `disable_functions` in php.ini is the right layer to block RCE-enabling functions like `system()`, `exec()`, `passthru()`
- Apache must be restarted after every php.ini change or the new config is not picked up
- The error log is the ground truth for confirming a function block is active — don't assume, verify
- Hardening goal is detection and delay, not absolute prevention — logs become more valuable after hardening, not less

## Gotchas
- Editing the CLI php.ini instead of the Apache one has zero effect on web requests — `find` returns both, pick carefully
- `disable_functions` only blocks PHP-land — if the app spawns processes via other vectors (e.g. ImageMagick, curl binary), the block is bypassed
- `open_basedir = /var/www` in php.ini locks PHP to the webroot and is a separate, complementary control worth adding


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-inclusion]]  
← [[09-automated-scanning]] | [[skills-assessment]] →
<!-- AUTO-LINKS-END -->
