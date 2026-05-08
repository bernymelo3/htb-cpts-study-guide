## ID
100

## Module
SQL Injection Fundamentals

## Kind
notes

## Title
Section 1 — Introduction

## Description
An introduction to SQL injection, its impact, and prevention, laying the foundation for the module.

## Tags
theory, sql-injection, web-app, introduction, mysql

## TL;DR — What's Important
- **SQL injection is user‑controlled SQL:** Any time user input reaches a database query, an attacker may be able to modify the query’s logic.
- **The core trick is escaping the data context:** Injecting a single quote (`'`) or double quote (`"`) allows breaking out of the intended string and inserting raw SQL.
- **Impact ranges from data theft to full server takeover:** Successful injection can steal credentials, bypass authentication, read/write server files, and eventually execute commands on the host.
- **Prevention is at the application and database layers:** Input validation, parameterised queries, and least‑privilege database accounts are the primary defences.
- **This module focuses on MySQL:** Relational database concepts are universal, but syntax examples will be MySQL‑oriented.

## Concept Overview
SQL injection (SQLi) is a vulnerability that arises when user‑supplied data is incorporated into a SQL query without proper sanitisation. An attacker crafts input that alters the structure of the query, enabling them to read, modify, or delete data, bypass application logic, or issue operating‑system commands. While the concept applies to any relational database, this module uses MySQL to demonstrate the attacks.

## Key Concepts

### Definitions
- **SQL Injection (SQLi)** — A code injection technique that exploits a security vulnerability in an application’s database layer by inserting malicious SQL statements into an entry field.
- **Relational Database** — A structured collection of data organised into tables that can be queried with SQL (e.g., MySQL, PostgreSQL, MSSQL). NoSQL databases are not covered here.
- **Stacked Queries** — Executing multiple SQL statements in a single call by separating them with a semicolon (`;`).
- **Union Query** — Using the `UNION` SQL operator to combine the results of two or more `SELECT` statements into a single result set, often used to extract data from other tables.

## Why It Matters
SQL injection remains one of the most critical web application vulnerabilities. Even a single vulnerable parameter can expose an entire database, lead to credential compromise, or provide an entry point for lateral movement inside a network. Understanding how SQLi works is essential for both offensive testers (finding and exploiting the flaw) and developers/defenders (preventing it).

## Defender Perspective
- **Detection:** Look for SQL keywords (`UNION`, `SELECT`, `OR 1=1`, etc.) in HTTP parameters, error messages that leak database details, and unusual database traffic patterns.
- **Mitigations:** Use parameterised queries (prepared statements) server‑side; never concatenate user input directly into SQL. Enforce strict allow‑list input validation. Apply the principle of least privilege to database accounts the web application uses (e.g., read‑only unless writes are required, no file/command privileges).
- **MITRE ATT&CK:** Primarily maps to *T1190 – Exploit Public‑Facing Application* and *T1059 – Command and Scripting Interpreter* (when code execution is achieved).

## Key Takeaways
- SQLi is not a single technique but a family of attacks; the simplest begins with a quote character.
- The ability to *inject* by itself is useless unless the attacker can reshape the query into something that executes and returns results.
- Modern frameworks often protect against basic injection, but the skill remains vital for testing legacy or custom‑built applications.
- Effective prevention requires defence‑in‑depth: secure coding, input validation, and strict database permissions.

## Gotchas
- Many “SQL injection” tools will report a vulnerability based solely on a generic error message; always confirm with a proof‑of‑concept query.
- Parameterised queries prevent injection in the data context but do not protect against injection into table or column names if the application dynamically builds those parts (rare, but possible).
- File read/write capability via SQL (e.g., `LOAD_FILE`, `INTO OUTFILE`) depends on the database user’s file privileges—always check before assuming you can pivot to RCE.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sql-injection-fundamentals]]  
[[02-intro-to-databases]] →
<!-- AUTO-LINKS-END -->
