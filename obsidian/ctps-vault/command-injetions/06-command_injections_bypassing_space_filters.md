## ID
531

## Module
Command Injections

## Kind
notes

## Title
Section 6 — Bypassing Space Filters

## Description
Demonstrates multiple techniques to bypass space character blacklists in command injection payloads, including tabs, $IFS, and brace expansion.

## Tags
command-injection, filter-evasion, space-bypass, ifs, brace-expansion, tabs

## Commands
- `127.0.0.1%0a%09ls%09-la` — tab (`%09`) as space replacement
- `127.0.0.1%0a${IFS}ls${IFS}-la` — Linux `$IFS` variable (defaults to space+tab)
- `127.0.0.1%0a{ls,-la}` — Bash brace expansion (auto-inserts spaces between args)

## What This Section Covers
When a web application blacklists the space character (common for IP-only input fields), you can still inject commands by substituting spaces with tabs (`%09`), the `$IFS` environment variable, or Bash brace expansion. All three are valid shell syntax that Linux interprets identically to spaces between arguments.

## Methodology
1. Confirm `%0a` (newline) still works as the injection operator from the previous section.
2. Test adding a space after the operator — `127.0.0.1%0a+whoami` → "Invalid input" confirms space is blacklisted.
3. Replace the space with one of these bypasses:
   - **Tab (`%09`):** `127.0.0.1%0a%09whoami` — URL-encoded horizontal tab; works on both Linux and Windows.
   - **`${IFS}`:** `127.0.0.1%0a${IFS}whoami` — Internal Field Separator; default value is space+tab+newline.
   - **Brace expansion:** `127.0.0.1%0a{ls,-la}` — Bash expands `{a,b}` to `a b` automatically.
4. For commands with multiple arguments (e.g., `ls -la`), replace every space: `127.0.0.1%0a%09ls%09-la`.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Execute `ls -la`. What is the size of `index.php`? | (answered correctly) | Used `127.0.0.1%0a%09ls%09-la` or `127.0.0.1%0a{ls,-la}` in Burp Repeater to list directory contents |

## Key Takeaways
- Space is one of the most commonly blacklisted characters — always have bypass techniques ready.
- `%09` (tab) is the simplest and most reliable space replacement; works on both Linux and Windows.
- `${IFS}` is Linux-only but powerful because it's a shell variable, not a literal character — filters checking for whitespace won't catch it.
- Brace expansion (`{cmd,arg}`) is elegant for single commands but gets awkward for complex chains.
- PayloadsAllTheThings has an extensive list of additional space bypass methods worth bookmarking.

## Gotchas
- `$IFS` without braces can sometimes cause parsing issues — prefer `${IFS}` with curly braces for reliability.
- Brace expansion only works in Bash — if the target shell is `sh` or `dash`, it won't expand.
- Remember to replace **every** space in the payload, not just the first one — `ls -la` needs two substitutions (`ls%09-la`).
