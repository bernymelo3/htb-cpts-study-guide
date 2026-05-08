# Section 4 — Intro to MySQL

## ID
530

## Module
SQL Injection Fundamentals

## Kind
notes

## Title
Section 4 — Intro to MySQL

## Description
Covers MySQL/MariaDB basics: connecting via the `mysql` CLI client, creating databases and tables, column data types, and table properties like AUTO_INCREMENT, NOT NULL, UNIQUE, DEFAULT, and PRIMARY KEY.

## Tags
mysql, sql, database, create-table, mariadb, cli

## Commands
- mysql -u root -p
- mysql -u root -h <TARGET_IP> -P <PORT> -p
- CREATE DATABASE <DB_NAME>;
- SHOW DATABASES;
- USE <DB_NAME>;
- CREATE TABLE <TABLE_NAME> (<COL> <TYPE>, ...);
- SHOW TABLES;
- DESCRIBE <TABLE_NAME>;

## What This Section Covers
Introduction to MySQL/MariaDB and the SQL language fundamentals needed to understand SQL injection. Walks through authenticating to a MySQL instance from the command line, creating databases and tables, and using column constraints and properties. This is the foundation for every SQLi technique covered later in the module.

## Methodology
1. Connect to the remote MySQL instance with `mysql -u root -h <IP> -P <PORT> -p`
2. List databases with `SHOW DATABASES;`
3. Switch to a database with `USE <DB_NAME>;`
4. List tables with `SHOW TABLES;` and inspect structure with `DESCRIBE <TABLE>;`
5. Create tables using `CREATE TABLE` with appropriate column types and constraints

## Key Takeaways
- SQL statements are case-insensitive (`USE` = `use`), but database/table **names** are case-sensitive
- Every MySQL CLI query must end with a semicolon `;`
- `-p` (lowercase) = password, `-P` (uppercase) = port — mixing these up is a classic mistake
- Never pass the password directly on the command line (`-p<pass>`) — it gets stored in `.bash_history`
- Default MySQL port is **3306** but HTB labs frequently use non-standard ports
- `NOT NULL` = required field; `UNIQUE` = no duplicates; `AUTO_INCREMENT` = auto-assigned IDs; `DEFAULT` = fallback value
- `PRIMARY KEY` uniquely identifies each row — critical concept for relational DB logic and later SQLi exploitation

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Name of the first database? | *(confirm from your SHOW DATABASES output)* | `mysql -u root -h <IP> -P 31637 -p` → `SHOW DATABASES;` → first entry in the list |


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sql-injection-fundamentals]]  
← [[03-types-of-databases]] | [[05-sql-statements]] →
<!-- AUTO-LINKS-END -->
