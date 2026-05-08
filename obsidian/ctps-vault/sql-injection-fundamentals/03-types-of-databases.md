## ID
102

## Module
SQL Injection Fundamentals

## Kind
notes

## Title
Section 3 — Types of Databases

## Description
Explains the two main database categories—relational and non-relational—their structures, use cases, and why SQL injection targets relational databases.

## Tags
theory, databases, relational-database, nosql, mysql, mongodb

## TL;DR — What's Important
- **Relational databases use schemas, tables, keys, and SQL:** Data is structured into rows and columns, linked by keys (e.g., `id` → `user_id`), forming a schema.
- **NoSQL databases are schema‑less and use different storage models:** Key‑Value, Document‑Based, Wide‑Column, and Graph. They do not use traditional SQL.
- **SQL injection applies only to relational databases:** NoSQL databases have their own injection class (NoSQL injection), which is a separate topic.
- **Tables are linked by keys, making complex queries efficient:** The `id` column in one table becomes a foreign key in another, enabling joins without duplicating data.
- **MySQL is the relational DBMS this module focuses on:** The same relational concepts (keys, schemas, SQL) apply across other RDBMS like PostgreSQL, MSSQL, and Oracle.

## Concept Overview
Databases fall into two broad categories: relational and non‑relational. Relational databases organise data into tables with predefined schemas and relationships established through keys. Non‑relational (NoSQL) databases abandon the table/row/column structure in favour of flexible storage models suited to unstructured or rapidly changing data. Understanding the distinction matters because injection attacks differ fundamentally between the two—this module deals exclusively with SQL injection in relational databases.

## Key Concepts

### Relational Databases
- **Schema** — The blueprint that defines tables, columns, data types, and relationships. All data must conform to it.
- **Table (Entity)** — A collection of related data stored in rows and columns (e.g., `users`, `posts`, `comments`).
- **Key** — A column used to link tables. A **primary key** uniquely identifies a row in its own table; a **foreign key** references the primary key of another table.
- **RDBMS (Relational Database Management System)** — Software that manages relational databases (MySQL, PostgreSQL, MSSQL, Oracle).
- **Example relationship:** `users.id` (primary key) ↔ `posts.user_id` (foreign key). A single query can join these tables to retrieve a post together with its author's details.

### Non‑relational Databases (NoSQL)
| Storage Model | Description |
|---------------|-------------|
| Key‑Value | Data stored as key:value pairs, often in JSON/XML. Like a dictionary. |
| Document‑Based | Similar to Key‑Value but values are structured documents (e.g., JSON objects) with queryable fields. |
| Wide‑Column | Data stored in columns rather than rows, optimised for large‑scale queries over sparse data. |
| Graph | Entities stored as nodes and relationships as edges, optimised for interconnected data. |

## Why It Matters
SQL injection exploits the structured, language‑driven nature of relational databases. The same keys and joins that make relational databases fast and logical also mean an attacker who can inject SQL can traverse the entire schema—reading from any table they can reach by following foreign keys. NoSQL databases present a different attack surface entirely, so correctly identifying the back‑end database type ensures you use the right injection approach.

## Defender Perspective
- **Detection:** Relational DBMS logs will show SQL syntax that includes keywords like `JOIN`, `UNION`, and `FROM` followed by system table names. NoSQL databases lack these patterns but can show injection in JSON or XML payloads (e.g., `$gt`, `$ne` operators in MongoDB).
- **Mitigations:** Understanding the schema allows DBAs to revoke unnecessary cross‑table access and enforce least privilege. For example, a web user account shouldn't be able to `SELECT` from `mysql.user` or `information_schema` tables unless required.
- **MITRE ATT&CK:** Knowledge of the database type and schema aids in *T1082 – System Information Discovery* and *T1003 – OS Credential Dumping* when databases store sensitive data.

## Key Takeaways
- The schema is both a strength and a weakness: it enables efficient queries but also maps the attack surface for an attacker who gains SQL injection.
- A foreign key relationship means you can often pivot from one table to another—once you're in the database, the relationships guide your exploration.
- NoSQL is not inherently more secure; it simply has a different injection methodology. Don't assume "NoSQL" means "no injection."
- The JSON representation of NoSQL data closely mirrors how it's stored and queried, making NoSQL injection payloads syntactically different from SQLi payloads.

## Gotchas
- Don't try SQL injection against a NoSQL database—you'll waste time and generate noise. Fingerprint the database type first (error messages, response behaviour, open ports).
- Some modern applications use both: a relational DB for structured transactional data and a NoSQL store for logs, sessions, or unstructured content. Injection might exist in one but not the other.
- Keys are rarely called just `id` in real‑world schemas—expect names like `user_id`, `uid`, `pk_user`, etc. Don't rely on guessing generic column names alone.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sql-injection-fundamentals]]  
← [[02-intro-to-databases]] | [[04-intro-to-mysql]] →
<!-- AUTO-LINKS-END -->
