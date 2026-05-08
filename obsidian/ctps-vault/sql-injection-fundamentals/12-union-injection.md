## ID
533

## Module
SQL Injection Fundamentals

## Kind
notes

## Title
Section 12 — Union Injection

## Description
Exploit union-based SQL injection by detecting column count, identifying visible columns, and injecting queries like `user()` or `@@version` to extract data from the database.

## Tags
sqli, union-injection, order-by, column-detection, data-extraction

## Commands
- ' order by 1-- -
- ' order by <N>-- -
- ' UNION select 1,2,3,4-- -
- ' UNION select 1,user(),3,4-- -
- ' UNION select 1,@@version,3,4-- -

## What This Section Covers
Union injection is the practical exploitation of UNION in a vulnerable web app. The workflow is three phases: detect how many columns the original query returns, figure out which of those columns are displayed on the page, then place your payload in a visible column. This lets you extract arbitrary data from the database through the app's own output.

## Methodology
1. **Detect column count (ORDER BY)** — Inject `' order by 1-- -`, increment the number until you get an error. The last successful number = total columns.
2. **Detect column count (UNION alternative)** — Inject `' UNION select 1,2,3-- -`, keep adding numbers until the query succeeds instead of erroring.
3. **Identify visible columns** — Once UNION works (e.g. `' UNION select 1,2,3,4-- -`), see which numbers appear on the page. Those are your injection points.
4. **Inject your payload** — Replace a visible column number with your query: `' UNION select 1,user(),3,4-- -`. The result shows up where that number was.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Use Union injection to get result of user() | *(paste answer here)* | `' UNION select 1,user(),3,4-- -` in search field (adjust column count as needed) |

## Key Takeaways
- ORDER BY method: works until it breaks (increment until error). UNION method: breaks until it works (add columns until success). Both give you the column count.
- Not all columns are displayed on the page — using junk numbers (1, 2, 3...) lets you see which positions are visible.
- Column 1 is often an ID that isn't rendered, so typically inject in column 2, 3, or 4.
- Useful test payloads once you have injection position: `user()`, `@@version`, `database()`.

## Gotchas
- If you use ORDER BY and the page just goes blank (no error), that also means the column doesn't exist — don't wait for an explicit error message.
- The search box might need a valid-looking prefix before the injection (e.g. `cn' UNION...`) or might work with just `' UNION...`. Try both.
- Remember the space after `--` : always use `-- -` or `--+`.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sql-injection-fundamentals]]  
← [[11-union-clause]] | [[13-database-enumeration]] →
<!-- AUTO-LINKS-END -->
