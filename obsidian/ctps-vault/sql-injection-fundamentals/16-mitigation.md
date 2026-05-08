## ID
104

## Module
SQL Injection Fundamentals

## Kind
notes

## Title
Section 16 — Mitigating SQL Injection

## Description
Covers multiple practical techniques to prevent SQL injection in web applications, including input sanitisation, validation, least‑privilege database accounts, WAFs, and parameterised queries.

## Tags
theory, mitigation, defence, sql-injection, parameterized-queries, waf

## TL;DR — What's Important
- **Sanitise input with dedicated functions:** `mysqli_real_escape_string()` escapes special characters like quotes, but it is not a complete defence on its own.
- **Validate input against an allow‑list pattern:** Use regular expressions (e.g., `/^[A-Za-z\s]+$/`) to reject anything that doesn’t match the expected format.
- **Apply least privilege at the database level:** The web application’s database user should only have the permissions it strictly needs (e.g., `SELECT` on specific tables).
- **Deploy a Web Application Firewall (WAF) as an additional layer:** WAFs can block known malicious patterns (e.g., `INFORMATION_SCHEMA`) even before they reach the application.
- **Prefer parameterised queries (prepared statements):** They separate SQL logic from data, eliminating injection entirely by design.
- **No single defence is foolproof:** Layering these techniques provides defence‑in‑depth against SQL injection.

## Concept Overview
Mitigating SQL injection requires a combination of secure coding practices, input handling, database configuration, and external protective layers. The goal is to ensure that user‑supplied data can never be interpreted as SQL code. This section examines five practical defences: input sanitisation with escaping functions, strict allow‑list input validation, database user privilege restriction, Web Application Firewalls, and parameterised queries (prepared statements).

## Key Concepts

### Input Sanitisation
Sanitisation removes or neutralises special characters that could alter the SQL syntax. The PHP function `mysqli_real_escape_string()` escapes characters such as `'` and `"` so they are treated as literal characters rather than string delimiters.
```php
$username = mysqli_real_escape_string($conn, $_POST['username']);
$password = mysqli_real_escape_string($conn, $_POST['password']);

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sql-injection-fundamentals]]  
← [[15-writing-files]] | [[skills-assessment]] →
<!-- AUTO-LINKS-END -->
