## ID
700

## Module
Command Injections

## Kind
notes

## Title
Section 1 — Intro to Command Injections

## Description
Definition, categories, and real‑world code examples of OS command injection vulnerabilities, explaining how unsanitised user input in system commands leads to complete server compromise.

## Tags
theory, concept, command-injection, injection-types, owasp

## TL;DR — What's Important
- **Command injection is a top‑tier risk:** It lets attackers run arbitrary OS commands on the server, often leading straight to full system takeover.
- **Injection = misplaced trust:** Any user input that becomes part of a command, query, or evaluated code without sanitisation is an injection opportunity.
- **Not just one language:** PHP (`system`, `exec`, `shell_exec`), NodeJS (`child_process.exec`), Python (`os.system`), and others all have equivalent dangerous functions.
- **Impact transcends web:** A command injection on a web server can pivot into the internal network, compromise databases, and escalate privileges.
- **Defenders must assume all input is malicious** and apply input validation, escaping, and the principle of least privilege to all execution paths.

## Concept Overview
Injection vulnerabilities occur when user‑controlled data is misinterpreted as part of a command or query. OS command injection is the specific instance where an attacker’s payload is passed to a function that runs system shell commands. The input is meant to be just a filename or a parameter, but without proper sanitisation or escaping, an attacker can break out of the intended context and inject their own commands. This section defines the injection family (command, code, SQL, XSS, etc.) and illustrates how vulnerable PHP and NodeJS code looks, setting the stage for practical exploitation.

## Key Concepts

### The Injection Family
| Injection Type | What Gets Misinterpreted | Typical Context |
|---|---|---|
| OS Command Injection | Shell command (e.g., `touch`, `ping`) | `system()`, `exec()`, backticks |
| Code Injection | Code evaluated at runtime (e.g., `eval()`) | `eval()`, `assert()` |
| SQL Injection | Database query (SQL) | `SELECT ... WHERE name = '$input'` |
| Cross‑Site Scripting (XSS) | HTML/JavaScript context | Reflected or stored output in a web page |

### How Command Injection Happens
- **PHP:** Functions like `system()`, `exec()`, `shell_exec()`, `passthru()`, `popen()` accept a string that is passed to the operating system’s shell.
- **NodeJS:** `child_process.exec()` and `child_process.spawn()` run commands; template literals (` `touch /tmp/${req.query.filename}.txt` `) are dangerous if user input is interpolated.
- **Other languages:** Python (`os.system`, `subprocess.Popen`), Ruby (backticks, `%x{}`), etc. – the pattern is the same.

### Simple Vulnerable Code Example (PHP)
```php
if (isset($_GET['filename'])) {
    system("touch /tmp/" . $_GET['filename'] . ".pdf");
}