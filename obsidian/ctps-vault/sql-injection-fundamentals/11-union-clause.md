## ID
532

## Module
SQL Injection Fundamentals

## Kind
notes

## Title
Section 11 — Union Clause

## Description
Understand the SQL UNION operator for combining results from multiple SELECT statements, including column-count matching and junk-data padding for uneven tables.

## Tags
sqli, union, column-matching, junk-data, union-clause

## Commands
- SELECT * FROM <TABLE1> UNION SELECT * FROM <TABLE2>;
- SELECT * FROM <TABLE1> UNION SELECT col1, col2, 3, 4, 5, 6 FROM <TABLE2>;
- SELECT COUNT(*) FROM (SELECT * FROM <TABLE1> UNION SELECT col1, col2, 3, 4, 5, 6 FROM <TABLE2>) AS combined;

## What This Section Covers
The UNION operator combines results from multiple SELECT statements into one result set. For SQL injection, this means you can append your own SELECT to the original query and pull data from any table in the database. The catch is both SELECTs must have the same number of columns — if they don't, you pad with junk numbers or NULL.

## Methodology
1. **Understand the rule** — UNION requires equal column counts on both sides. Mismatched columns throw an error.
2. **Check column counts** — Use `DESCRIBE <table>` to see how many columns each table has.
3. **Pad with junk data** — If table A has 6 columns and table B has 2, select from B with 4 junk values: `SELECT col1, col2, 3, 4, 5, 6 FROM tableB`.
4. **Use NULL for safety** — `NULL` fits all data types, so it's the safest padding choice for advanced injections.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: UNION employees + departments, count rows | *(paste count here)* | `SELECT COUNT(*) FROM (SELECT * FROM employees UNION SELECT dept_no, dept_name, 3, 4, 5, 6 FROM departments) AS combined;` |

## Key Takeaways
- UNION removes duplicate rows; use `UNION ALL` if you want duplicates preserved.
- Both SELECTs must return the same number of columns or you get error 1222.
- Pad with numbers (1, 2, 3...) for tracking which column is which, or NULL for type safety.
- Data types should match across positions — numbers and NULL are the safest fillers.

## Gotchas
- Forgetting to pad columns is the #1 error — always check column counts first with `DESCRIBE`.
- UNION deduplicates by default, so your row count may be less than the sum of both tables.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sql-injection-fundamentals]]  
← [[10-using-comments]] | [[12-union-injection]] →
<!-- AUTO-LINKS-END -->
