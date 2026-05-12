# LAB — Recursive Fuzzing

## ID
5

## Module
Ffuf

## Kind
lab

## Title
Section 5 — Recursive Fuzzing

## Description
One ffuf run that fuzzes directories and automatically descends into discovered subdirs, with `-e` to also try a fixed file extension at every level.

## Tags
ffuf, recursive-fuzzing, lab, directory-tree

## Commands
- `ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://SERVER_IP:PORT/FUZZ -recursion -recursion-depth 1 -e .php -v`
- `ffuf -s -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u 'http://STMIP:STMPO/FUZZ' -recursion -recursion-depth 1 -e '.php'`

## What This Section Covers
`-recursion` queues a new fuzz job under every directory ffuf finds. Without depth limits, a deep tree can cascade out of control. `-recursion-depth 1` limits recursion to one level below the start URL. `-e .php` appends an extension list so each candidate is tried both bare (directory) and with `.php` (file).

## Methodology
1. Place `FUZZ` once in the URL path: `-u http://target/FUZZ`.
2. Add `-recursion` to enable subdir descent.
3. Cap depth: `-recursion-depth 1` (or 2 if the app is deep).
4. Add extensions: `-e .php` (or `-e .php,.html,.aspx`).
5. Add `-v` so the output prints full URLs — without it, identical filenames in different subdirs are ambiguous.
6. Watch the request count: depth × extensions × wordlist size.

## Recursion Math
- Wordlist of 87 k × 1 extension = ~174 k requests at depth 0.
- Each new directory found adds another full scan at that level.
- Depth 2 on a wide site can run for hours — start with depth 1.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Flag from recursive fuzzing the root | **HTB{fuzz1n6_7h3_w3b!}** | `ffuf -s -w directory-list-2.3-small.txt:FUZZ -u 'http://STMIP:STMPO/FUZZ' -recursion -recursion-depth 1 -e '.php'` revealed `/forum/flag.php` |

## Key Takeaways
- Recursion replaces "fuzz, find, cd, fuzz again" with a single command — but it's only safe with a depth cap.
- Always pair `-recursion` with `-v` so the output is readable.
- Extensions are site-wide on most apps — `-e .php` carries through every depth level for free.

## Gotchas
- ffuf recursion **does not** retroactively re-fuzz parents when a child is found — depth is fixed at start.
- A misconfigured app may serve every URL with `200 OK`; without filtering you'll fuzz the whole tree forever.
- Recursion + extensions doubles the wordlist size at every level — a 4-extension list on depth 2 is 8× the requests.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ffuf]]
← [[04-page-fuzzing]] | [[06-dns-records]] →
<!-- AUTO-LINKS-END -->
