## ID
300

## Module
SQLMap Essentials

## Kind
notes

## Title
Section 1 — SQLMap Overview

## Description
Introduces SQLMap, its capabilities, supported databases, and the full range of SQL injection techniques it automates.

## Tags
sqlmap, sql-injection, tool-overview, theory, enumeration, exploitation

## TL;DR — What's Important
- **SQLMap is the leading open‑source tool for automated SQL injection detection and exploitation**, written in Python and actively maintained since 2006.
- **It supports an extensive range of DBMSes** (MySQL, PostgreSQL, Oracle, MSSQL, SQLite, and many more).
- **Six core SQLi techniques are represented by the mnemonic BEUSTQ:**  
  - **B**oolean‑based blind, **E**rror‑based, **U**nion query‑based, **S**tacked queries, **T**ime‑based blind, Inline **Q**ueries.
- **Union and Error‑based are the fastest**; Boolean‑based blind and Time‑based blind are slower but essential when no output is visible.
- **SQLMap can also perform out‑of‑band (OOB) exfiltration** via DNS when other methods fail.
- **The tool can read/write files, execute OS commands, and bypass WAFs** using tamper scripts.
- **Installation is trivial:** available via `apt` on most pentest distributions or clone from GitHub.

## Concept Overview
SQLMap automates the entire SQL injection lifecycle — from identifying injectable parameters to extracting database contents, gaining file‑system access, or even executing operating‑system commands. It uses a powerful detection engine that profiles target responses (content, HTTP codes, timing) to determine the presence and type of injection, then applies the appropriate exploitation technique. Understanding its capabilities and limitations allows a penetration tester to quickly assess and exploit vulnerable web applications.

## Key Concepts

### Injection Techniques (BEUSTQ)
| Technique | Character | Speed | Typical Scenario |
|-----------|-----------|-------|------------------|
| **Boolean‑based blind** | B | Medium (~7–8 requests/char) | Page content changes with true/false condition; common in web apps. |
| **Error‑based** | E | Fast (chunks up to 200 bytes) | Database errors are displayed on the page; second fastest after Union. |
| **Union query‑based** | U | Fastest (entire tables in single request) | Original query results are rendered; ideal when possible. |
| **Stacked queries** | S | Varies | Need to run non‑query statements (`INSERT`, `DELETE`); supported by MSSQL, PostgreSQL by default. |
| **Time‑based blind** | T | Slow (delay per true condition) | No visible difference except response time; used when Boolean‑blind won’t work (e.g., non‑query context). |
| **Inline queries** | Q | Depends | Embedded sub‑select inside the original query; uncommon. |

SQLMap also supports **Out‑of‑band (OOB) SQLi** via DNS exfiltration, forcing the server to resolve attacker‑controlled subdomains that carry the data.

### Installation
- Pre‑installed on PwnBox, Kali, Parrot.
- Debian: `sudo apt install sqlmap`
- Manual: `git clone --depth 1 https://github.com/sqlmapproject/sqlmap.git sqlmap-dev` then run with `python sqlmap.py`

### Supported Databases
Extensive list including MySQL, PostgreSQL, Oracle, MSSQL, SQLite, DB2, Firebird, Sybase, Access, MariaDB, CockroachDB, and many more.

## Why It Matters
Manual SQL injection is time‑consuming and error‑prone, especially when large amounts of data need to be extracted. SQLMap shrinks hours of work into minutes, reliably handles encoding, and can pivot from database access to OS command execution. Knowing what each technique does and when to use (or disable) certain options helps tailor scans for speed and stealth.

## Defender Perspective
- **Detection:** SQLMap’s requests often contain characteristic patterns like `sqlmap` user‑agent, `--technique` payloads, or rapid requests with varying parameters. WAFs and IDS can detect these.
- **Mitigations:** The same as for SQL injection generally — parameterised queries, input validation, least privilege. Tamper scripts can bypass some WAF rules, so WAF signatures must be regularly updated.
- **MITRE ATT&CK:** Maps to *T1190 – Exploit Public‑Facing Application*; also *T1059 – Command and Scripting Interpreter* when OS command execution is achieved.

## Key Takeaways
- SQLMap is not a magic wand; a basic understanding of SQLi is still required to interpret results and handle edge cases.
- The `--technique` flag allows disabling certain techniques to speed up testing or avoid detection.
- Boolean‑based blind can extract data character by character without heavy time‑based delays, making it stealthier than Time‑based.
- Error‑based injection is underrated — when database errors are revealed, data extraction is lightning fast.
- Always check legal disclaimers: SQLMap prints a warning; the user is responsible for obtaining proper authorisation.

## Gotchas
- Default `sqlmap` command can generate extensive noise; fine‑tune with `--risk` and `--level` or risk detection and potential blocking.
- Some servers respond differently to a valid vs. invalid SQL condition without any visible change; Boolean‑based blind may still work by analysing minute differences (e.g., line counts, response hashing) that SQLMap does automatically.
- Stacked queries are not supported by MySQL when using the `query()` function in PHP, but may work if the application uses `multi_query()`.
- Time‑based blind is accurate but extremely slow over high‑latency connections or when extracting large amounts of data; consider Out‑of‑band if possible.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sqlmap-fundamentals]]  
[[02-getting-started]] →
<!-- AUTO-LINKS-END -->
