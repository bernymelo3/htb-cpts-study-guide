## ID
103

## Module
SQL Injection Fundamentals

## Kind
notes

## Title
Section 8 — Intro to SQL Injections

## Description
Defines SQL injection, shows how unsanitised user input becomes code, explains syntax errors, and categorises injection types (In‑band, Blind, Out‑of‑band).

## Tags
theory, sql-injection, union-based, error-based, blind-sqli, injection-types

## TL;DR — What's Important
- **Injection turns input into code:** When user data is placed directly into a SQL string without sanitisation, an attacker can break out of the data context and inject arbitrary SQL.
- **The single quote is the classic escape character:** Injecting `'` closes the string literal and allows raw SQL to be appended (e.g., `1' OR '1'='1`).
- **A successful injection must keep the final query syntactically valid:** Syntax errors often appear when leftover characters break the query; comments or additional quotes can resolve them.
- **SQLi categories dictate how you get the output back:** In‑band (results shown on the page), Blind (inferred from behaviour), Out‑of‑band (exfiltrated via external channels).
- **Stacked queries (`;`) are not always possible in MySQL:** The module focuses on `UNION`‑based injection as the primary in‑band technique.

## Concept Overview
SQL injection occurs when user input is inserted into a SQL query without proper sanitisation, allowing an attacker to alter the query’s structure. A web application written in PHP (or any language) builds a SQL string that includes unsanitised input; a malicious user can inject a single quote to escape the string context and add arbitrary SQL. To avoid syntax errors, the attacker must ensure the entire query remains valid, often by commenting out the remainder of the original query or by balancing quotes.

## Key Concepts

### How Injection Works (PHP Example)
```php
$searchInput = $_POST['findUser'];
$query = "SELECT * FROM logins WHERE username LIKE '%$searchInput'";
$result = $conn->query($query);

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sql-injection-fundamentals]]  
← [[07-sql-operators]] | [[09-subverting-query-logic]] →
<!-- AUTO-LINKS-END -->
