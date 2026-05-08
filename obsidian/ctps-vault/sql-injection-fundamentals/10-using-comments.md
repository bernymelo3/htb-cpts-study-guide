## ID
531

## Module
SQL Injection Fundamentals

## Kind
notes

## Title
Section 10 — Using Comments

## Description
Use SQL comments (`-- -` and `#`) to chop off the rest of a query after your injection point, bypassing parentheses, hashed passwords, and additional WHERE conditions.

## Tags
sqli, comments, authentication-bypass, parentheses-escape, comment-injection

## Commands
- admin'-- -
- admin')-- -
- ') or id=<ID> -- -
- ' or 1=1 -- -
- %23 (URL-encoded #)

## What This Section Covers
SQL comments let you kill everything after your injection point, so the database never sees the password check or any other conditions. This is more powerful than the OR approach from Section 9 because you don't need to worry about balancing quotes or operator precedence — you just cut the query short. Essential when passwords are hashed server-side (can't inject through the password field) or when parentheses add extra conditions like `id > 1`.

## Methodology
1. **Simple comment bypass** — Use `admin'-- -` as the username. This closes the username string and comments out the password check entirely. The DB only sees `WHERE username='admin'`.
2. **Parentheses-aware bypass** — If the query wraps conditions in parentheses like `(username='X' AND id > 1)`, inject `admin')-- -`. The `)` closes the parenthesis, `-- -` kills the rest.
3. **Target a specific row by id** — If you don't know the username but know the id, inject `') or id=5 -- -`. This closes the parenthesis, adds an OR condition targeting the exact row, and comments out everything else.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Login as user with id 5, get the flag | *(paste flag here)* | Injected `') or id=5 -- -` in username field, any password |

## Key Takeaways
- Two comment styles in MySQL: `-- -` (needs a space after the dashes) and `#` (URL-encode as `%23` in browser).
- Comments let you ignore hashed password fields — if the app hashes your input before putting it in the query, you can't inject through password, so comment it out from the username side.
- Always match your parentheses — if the original query has `(` before your injection point, you need to close it with `)` before the comment, otherwise you get a syntax error.
- `-- ` alone isn't enough in MySQL; it needs a trailing space. Convention is `-- -` to make the space visible, or `--+` in URLs (+ = space).
- This technique builds on Section 9: use OR to pick your target row, then use comments to remove everything you don't control.

## Gotchas
- Forgetting the space after `--` causes a syntax error — always use `-- -` or `--+`.
- If the query has parentheses and you don't close them before commenting, the query breaks even though the comment is valid.
- `#` works in a MySQL terminal but in a browser URL it's treated as a page anchor — must use `%23` instead.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sql-injection-fundamentals]]  
← [[09-subverting-query-logic]] | [[11-union-clause]] →
<!-- AUTO-LINKS-END -->
