# Section 6 — Query Results

## ID
532

## Module
SQL Injection Fundamentals

## Kind
notes

## Title
Section 6 — Query Results

## Description
Covers controlling SELECT output with ORDER BY, LIMIT (with offsets), WHERE conditions, and LIKE pattern matching — all essential for crafting and understanding SQL injection payloads.

## Tags
mysql, sql, order-by, limit, where, like, wildcard

## Commands
- SELECT * FROM <TABLE> ORDER BY <COL>;
- SELECT * FROM <TABLE> ORDER BY <COL> DESC;
- SELECT * FROM <TABLE> ORDER BY <COL1> DESC, <COL2> ASC;
- SELECT * FROM <TABLE> LIMIT <COUNT>;
- SELECT * FROM <TABLE> LIMIT <OFFSET>, <COUNT>;
- SELECT * FROM <TABLE> WHERE <CONDITION>;
- SELECT * FROM <TABLE> WHERE <COL> LIKE '<PATTERN>';

## What This Section Covers
How to control and filter the output of SELECT queries. ORDER BY sorts results by one or more columns (ascending by default). LIMIT restricts the number of rows returned and supports offsets for pagination. WHERE filters rows by exact conditions, and LIKE enables pattern matching with wildcards. These clauses are heavily abused in SQL injection — ORDER BY is used to enumerate column counts in UNION attacks, and WHERE/LIKE filtering logic is what attackers manipulate to bypass authentication.

## Methodology
1. Connect to the target with `mysql -u root -h <IP> -P <PORT> -p`
2. `USE employees;` then `DESCRIBE employees;` to understand the table schema
3. Build a query combining `WHERE` with `LIKE` and an exact date match:
   `SELECT * FROM employees WHERE first_name LIKE 'Bar%' AND hire_date = '1990-01-01';`
4. Read the `last_name` column from the result

## Key Takeaways
- `ORDER BY` defaults to ascending (`ASC`); append `DESC` for descending — in SQLi, `ORDER BY <N>` is the classic way to find column count
- `LIMIT <offset>, <count>` — offset is **zero-indexed**, so `LIMIT 1, 2` skips the first row and returns the next two
- `WHERE` conditions: strings/dates must be quoted (`'value'`), numbers don't need quotes
- `LIKE` wildcards: `%` matches zero or more characters, `_` matches exactly one character
- You can combine multiple conditions with `AND` — this is the same logic attackers chain in WHERE-based SQLi
- `LIKE 'admin%'` would match `admin`, `administrator`, `admin_backup` etc. — important for understanding injection filter bypasses

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Last name of employee whose first name starts with "Bar" and hired on 1990-01-01? | Mitchem | `SELECT * FROM employees WHERE first_name LIKE 'Bar%' AND hire_date = '1990-01-01';` |

## Gotchas
- `LIMIT 1, 2` does NOT mean "limit to rows 1 and 2" — it means "skip 1 row, then return 2 rows" (offset, count)
- Forgetting quotes around string/date values in `WHERE` will throw a syntax error or produce unexpected results
- `_` in LIKE is a single-char wildcard — if you need a literal underscore, escape it with `\_`


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sql-injection-fundamentals]]  
← [[05-sql-statements]] | [[07-sql-operators]] →
<!-- AUTO-LINKS-END -->
