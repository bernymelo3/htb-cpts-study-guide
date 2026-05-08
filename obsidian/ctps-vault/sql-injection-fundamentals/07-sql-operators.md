# Section 7 — SQL Operators

## ID
533

## Module
SQL Injection Fundamentals

## Kind
notes

## Title
Section 7 — SQL Operators

## Description
Covers SQL logical operators (AND, OR, NOT) and their symbol equivalents (&&, ||, !), operator precedence, and how these are used in queries — foundational for understanding SQLi condition manipulation.

## Tags
mysql, sql, operators, and, or, not, precedence

## Commands
- SELECT * FROM <TABLE> WHERE <COL> != '<VALUE>';
- SELECT * FROM <TABLE> WHERE <COND1> AND <COND2>;
- SELECT * FROM <TABLE> WHERE <COND1> OR <COND2>;
- SELECT * FROM <TABLE> WHERE <COL> NOT LIKE '<PATTERN>';
- SELECT COUNT(*) FROM <TABLE> WHERE <CONDITION>;

## What This Section Covers
How SQL logical operators AND, OR, and NOT work and how they combine conditions in queries. These are the exact operators attackers inject to manipulate WHERE clauses — for example, `OR 1=1` to bypass authentication or `AND 1=2` to create false conditions for blind SQLi. Understanding operator precedence is critical because it determines how MySQL evaluates chained conditions, which directly affects payload behavior.

## Methodology
1. Connect to the target and `USE employees;`
2. Inspect the table with `DESCRIBE titles;`
3. Build a query combining `OR` and `NOT LIKE`:
   `SELECT COUNT(*) FROM titles WHERE emp_no > 10000 OR title NOT LIKE '%engineer%';`
4. The returned count is the answer

## Key Takeaways
- `AND` = both conditions must be true; `OR` = at least one must be true; `NOT` = inverts the boolean
- Symbol equivalents: `AND` = `&&`, `OR` = `||`, `NOT` = `!` — you'll see both forms in SQLi payloads
- **Operator precedence** (high → low): arithmetic → comparisons → NOT → AND → OR
- Because AND binds tighter than OR, `A OR B AND C` evaluates as `A OR (B AND C)` — this is exploited in auth bypass payloads like `' OR 1=1-- -`
- MySQL returns `1` for true and `0` for false — useful to know when interpreting blind SQLi responses
- `NOT LIKE '%engineer%'` excludes any title containing "engineer" anywhere in the string (case-insensitive)
- `COUNT(*)` returns the number of matching rows — commonly used in lab questions and in blind SQLi extraction

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Number of records where emp_no > 10000 OR title NOT LIKE '%engineer%'? | *(fill in from your COUNT result)* | `SELECT COUNT(*) FROM titles WHERE emp_no > 10000 OR title NOT LIKE '%engineer%';` |

## Gotchas
- `NOT LIKE` is case-insensitive by default in MySQL — don't worry about matching "Engineer" vs "engineer"
- Watch out for precedence: `WHERE A OR B AND C` is NOT `(A OR B) AND C` — AND evaluates first, so it's `A OR (B AND C)`. Use parentheses to be explicit
- The `!=` operator and `NOT` are different: `!=` is a comparison, `NOT` is a logical negation applied to an expression


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sql-injection-fundamentals]]  
← [[06-query-results]] | [[08-intro-to-sqli]] →
<!-- AUTO-LINKS-END -->
