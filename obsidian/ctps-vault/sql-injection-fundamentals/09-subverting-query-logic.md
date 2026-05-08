## ID
530

## Module
SQL Injection Fundamentals

## Kind
notes

## Title
Section 9 — Subverting Query Logic

## Description
Bypass web authentication by injecting OR conditions into login forms, exploiting SQL operator precedence (AND before OR) to force queries to return true.

## Tags
sqli, authentication-bypass, or-injection, operator-precedence, login-bypass

## Commands
- <USERNAME>' or '1'='1
- ' or '1'='1
- something' or '1'='1

## What This Section Covers
SQL injection against login forms by abusing the OR operator to subvert query logic. Because AND is evaluated before OR in SQL, injecting an OR true condition in the right field can force the WHERE clause to return rows regardless of credentials. This is the foundation of auth bypass attacks.

## Methodology
1. **Test for SQLi** — Inject a single quote `'` into the username field. If you get a SQL syntax error instead of "Login failed", the form is injectable.
2. **Target a specific user** — Use `tom' or '1'='1` as the username (any password). The query becomes `WHERE username='tom' or '1'='1' AND password='x'`. AND evaluates first: `'1'='1' AND password='x'` → False. Then `username='tom' OR False` → True (if tom exists). Only tom's row is returned.
3. **Blind login (no known user)** — Use `' or '1'='1` as username AND `something' or '1'='1` as password. This makes the entire WHERE clause true, returning all rows. You log in as the first user in the table.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Log in as tom, get the flag | *(paste flag here)* | Injected `tom' or '1'='1` in username field, any password |

## Key Takeaways
- AND is evaluated before OR in SQL — this is the core of why OR injection works for targeting specific users.
- Injecting OR into the **username** field targets a specific user; injecting into **both** fields returns all rows (first row = first user in table).
- Always test for SQLi first with `'`, `"`, `#`, `;`, `)` — a syntax error confirms injectability.
- The payload `admin' or '1'='1` keeps quotes balanced by relying on the existing closing quote in the original query.
- PayloadsAllTheThings has a comprehensive list of auth bypass payloads for different SQL query structures.

## Gotchas
- `notAdmin' or '1'='1` in the username field alone will **fail** if that user doesn't exist — the AND condition (password) still evaluates to false and OR with a non-existent user is still false overall.
- If you need to log in as a specific user but the OR-in-username approach isn't working, you may need to inject in both fields or use comments (next section).


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sql-injection-fundamentals]]  
← [[08-intro-to-sqli]] | [[10-using-comments]] →
<!-- AUTO-LINKS-END -->
