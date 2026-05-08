# Section 5 — SQL Statements

## ID
531

## Module
SQL Injection Fundamentals

## Kind
notes

## Title
Section 5 — SQL Statements

## Description
Covers the core CRUD SQL statements — INSERT, SELECT, DROP, ALTER, and UPDATE — which form the building blocks for understanding and crafting SQL injection payloads.

## Tags
mysql, sql, insert, select, update, alter, drop

## Commands
- INSERT INTO <TABLE> VALUES (<VAL1>, <VAL2>, ...);
- INSERT INTO <TABLE>(<COL1>, <COL2>) VALUES (<VAL1>, <VAL2>);
- SELECT * FROM <TABLE>;
- SELECT <COL1>, <COL2> FROM <TABLE>;
- UPDATE <TABLE> SET <COL>=<VALUE> WHERE <CONDITION>;
- ALTER TABLE <TABLE> ADD <COL> <TYPE>;
- ALTER TABLE <TABLE> RENAME COLUMN <OLD> TO <NEW>;
- ALTER TABLE <TABLE> DROP <COL>;
- DROP TABLE <TABLE>;

## What This Section Covers
Walks through the essential SQL statements for manipulating data and table structure. INSERT adds rows, SELECT retrieves data, UPDATE modifies existing rows conditionally, ALTER changes table schema, and DROP deletes tables entirely. Understanding these is critical because SQL injection fundamentally involves injecting or manipulating these statements.

## Methodology
1. Connect to the target MySQL instance with `mysql -u root -h <IP> -P <PORT> -p`
2. Enumerate databases with `SHOW DATABASES;` and switch with `USE <DB>;`
3. List tables with `SHOW TABLES;` and inspect with `DESCRIBE <TABLE>;`
4. Query for the answer with `SELECT * FROM departments WHERE dept_name = 'Development';`

## Key Takeaways
- `INSERT INTO` can skip columns that have defaults or `AUTO_INCREMENT` — just specify the columns you're filling
- You can bulk-insert multiple rows in one `INSERT` by comma-separating value tuples
- `SELECT *` grabs all columns; specifying column names is cleaner and mirrors what you'll do in UNION-based SQLi
- `UPDATE` without a `WHERE` clause updates **every row** — always include a condition
- `DROP TABLE` is permanent with no confirmation prompt — in an attack context, this is destructive and loud
- `ALTER TABLE` changes schema (add/rename/modify/drop columns) — useful to understand when reading DB structure during enumeration
- Cleartext passwords in tables are a bad practice but you'll encounter them constantly in CTFs and real-world breaches

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Dept number for 'Development'? | d005 | `USE employees;` → `SELECT * FROM departments WHERE dept_name = 'Development';` |

## Gotchas
- The `employees` database is pre-populated on the target — don't waste time looking for the `logins` table from the examples; it doesn't exist on this instance
- If you run `UPDATE` without `WHERE`, you'll modify every row in the table — no undo


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sql-injection-fundamentals]]  
← [[04-intro-to-mysql]] | [[06-query-results]] →
<!-- AUTO-LINKS-END -->
