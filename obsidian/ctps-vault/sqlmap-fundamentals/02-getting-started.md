## ID
301

## Module
SQLMap Essentials

## Kind
notes

## Title
Section 2 — Getting Started with SQLMap

## Description
Covers SQLMap’s help system, basic usage, and a first‑run example against a simple vulnerable GET parameter.

## Tags
sqlmap, help, usage, example, command-line

## Commands
- `sqlmap -h` — show basic help
- `sqlmap -hh` — show advanced help (all options)
- `sqlmap -u "http://www.example.com/vuln.php?id=1" --batch` — basic automated SQLi test with no user interaction

## What This Section Covers
You'll learn how to access SQLMap’s built‑in documentation (basic and advanced help), understand the common command structure, and see a real run against a vulnerable PHP page that uses a numeric GET parameter. The goal is to be able to launch SQLMap with minimal options and interpret its initial detection output.

## Methodology
1. Print the basic help: `sqlmap -h` – shows essential flags like `-u` (URL), `--data` (POST), `--cookie`, etc.
2. For detailed options, print advanced help: `sqlmap -hh` – includes all switches, injection techniques, request tuning, detection, etc.
3. Identify a target URL with a GET parameter, e.g., `http://www.example.com/vuln.php?id=1`.
4. Run SQLMap with the target and `--batch` to accept defaults automatically:
   ```bash
   sqlmap -u "http://www.example.com/vuln.php?id=1" --batch

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sqlmap-fundamentals]]  
← [[01-overview]] | [[03-output-description]] →
<!-- AUTO-LINKS-END -->
