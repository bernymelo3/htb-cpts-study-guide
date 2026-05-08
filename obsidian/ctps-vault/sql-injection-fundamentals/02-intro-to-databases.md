## ID
101

## Module
SQL Injection Fundamentals

## Kind
notes

## Title
Section 2 — Intro to Databases

## Description
Covers the fundamentals of databases and Database Management Systems (DBMS), their architecture, and why they matter for understanding SQL injection.

## Tags
theory, databases, dbms, architecture, mysql, introduction

## TL;DR — What's Important
- **A DBMS manages the database:** It handles concurrency, consistency, security, reliability, and provides a structured way to query data via SQL.
- **Web applications use a tiered architecture:** Client (Tier I) → Application/Middleware (Tier II) → DBMS (Tier III). SQL injection exploits the boundary between Tier II and Tier III.
- **User input flows through the middleware to the DBMS:** If the middleware doesn't sanitise inputs, malicious SQL reaches the database unaltered.
- **DBMS features are a double-edged sword:** The same SQL language that simplifies data access also enables powerful attacks if misused.
- **Understanding the architecture helps you map attack surfaces:** Knowing where queries are built and how they reach the database tells you where to probe for injection.

## Concept Overview
A database is an organised collection of data. A Database Management System (DBMS) is the software that creates, hosts, and manages databases — handling everything from concurrent user access to backup and recovery. Web applications typically sit in front of a DBMS, sending queries based on user input. When the application fails to validate that input, attackers can inject their own SQL, turning the DBMS from a data store into an attack vector.

## Key Concepts

### Definitions
- **Database** — A structured set of data. Can be file-based, relational (RDBMS), NoSQL, graph-based, or key/value.
- **DBMS (Database Management System)** — Software that manages databases, providing interfaces for creating, reading, updating, and deleting data.
- **Middleware** — The application layer (Tier II) that translates user actions (login, comment, search) into database queries, then processes the results back to the client.
- **SQL (Structured Query Language)** — The standard language for interacting with relational databases. Its intuitive syntax enables operations like `SELECT`, `INSERT`, `UPDATE`, and `DELETE`.

### DBMS Core Features
| Feature | Description |
|---------|-------------|
| Concurrency | Handles multiple simultaneous users without corrupting or losing data. |
| Consistency | Ensures data remains valid and coherent across all transactions. |
| Security | Enforces authentication and permissions to prevent unauthorised access. |
| Reliability | Supports backups and rollbacks to recover from failures or breaches. |
| SQL Support | Provides an intuitive syntax for users and applications to interact with data. |

### Architecture — Three Tiers
| Tier | Role | Example |
|------|------|---------|
| Tier I — Client | User-facing interface | Web browser, GUI application |
| Tier II — Middleware / App Server | Translates user actions into DBMS queries | PHP, Python, Node.js back-end |
| Tier III — DBMS | Executes queries and stores data | MySQL, PostgreSQL, MSSQL |

## Why It Matters
SQL injection sits at the boundary between Tier II and Tier III. The middleware builds a query using user input and sends it to the DBMS. If the input is trusted blindly, an attacker can reshape that query. Knowing how a DBMS operates — and that it will faithfully execute whatever syntactically valid SQL it receives — is the foundation for understanding why injection works and how far it can reach.

## Defender Perspective
- **Detection:** Monitor database query logs for unusual patterns (unexpected `UNION`, stacked queries, or access to system tables). Application firewalls can block generic SQLi patterns but may miss sophisticated injections.
- **Mitigations:** The DBMS itself can limit damage — use dedicated, low-privilege accounts for the web application; revoke file privileges and access to system tables where possible.
- **MITRE ATT&CK:** DBMS interaction maps to discovery techniques and credential access when databases store user hashes.

## Key Takeaways
- The DBMS is an obedient executor — it has no way to distinguish a legitimate query from one that has been tampered with. The responsibility for safe queries lies entirely with the middleware.
- Even if the application server and DBMS run on the same host, the logical separation between Tier II and Tier III remains where injection can occur.
- DBMS security features (fine-grained permissions) are a fallback, not a replacement for secure coding — once injection exists, the attacker may be able to escalate privileges within the database itself.
- File-based databases and NoSQL have different attack vectors — this module focuses on relational DBMS and SQL injection.

## Gotchas
- Not every dynamic web application uses a relational database — always confirm the backend technology before assuming SQL injection is the right attack path.
- A DBMS returning an "error" doesn't always mean no data was exposed — error messages can leak schema details, version info, or even data rows (error-based SQLi).
- The tiered architecture diagram shows logical separation; in smaller deployments, Tier II (app) and Tier III (DBMS) might coexist on the same server, but the data flow remains the same.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sql-injection-fundamentals]]  
← [[01-intro]] | [[03-types-of-databases]] →
<!-- AUTO-LINKS-END -->
