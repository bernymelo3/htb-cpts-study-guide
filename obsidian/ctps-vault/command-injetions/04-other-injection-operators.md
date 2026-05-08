# Section 4 — Other Injection Operators

## ID
602

## Module
Command Injections

## Kind
notes

## Title
Section 4 — Other Injection Operators

## Description
Explores AND, OR, new-line, background, and pipe injection operators, comparing their behavior and output differences when chaining commands.

## Tags
command-injection, operators, and, or, pipe, burp-suite

## Commands
- 127.0.0.1 && whoami
- 127.0.0.1 || whoami
- || whoami
- 127.0.0.1 | whoami
- 127.0.0.1 & whoami
- 127.0.0.1%0a whoami

## What This Section Covers
Different injection operators produce different behaviors — some run both commands, some conditionally run the second, and some only display output from one. Understanding these differences lets you pick the right operator depending on whether you need stealth, clean output, or guaranteed execution.

## Methodology
1. Test `&&` (AND) — both commands run, but only if the first succeeds. URL-encode as `%26%26` in Burp.
2. Test `||` (OR) — second command only runs if the first fails. Use `|| whoami` with no IP to force the first command to fail.
3. Test `|` (pipe) — both commands run, but only the second command's output is displayed.
4. Test `&` (background) — both commands run; second output generally appears first.
5. Test `%0a` (new-line) — both commands run as separate lines.
6. Always URL-encode payloads in Burp (`Ctrl+U`) before sending.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Which of the remaining three operators (new-line, &, \|) only shows the output of the injected command? | **\|** | Pipe sends the first command's stdout as stdin to the second, so only the second command's output is returned in the response |

## Key Takeaways
- `&&` requires the first command to succeed (exit code 0) — if the IP is invalid, the injected command won't run.
- `||` requires the first command to fail — useful when you can't supply valid input or want to skip the original command entirely.
- `|` is the cleanest for exfiltration — you only see your injected command's output, no ping noise.
- `&` backgrounds the first command, so both run but output ordering can be unpredictable.
- These same operator concepts apply across injection types (SQLi, LDAP, XPath, etc.) — not just OS command injection.

## Gotchas
- `||` with a valid IP before it will never trigger the second command — the first ping succeeds, so OR short-circuits.
- `&&` without a valid first command silently swallows the second — easy to think injection failed when it's just the operator logic.
- `&` must be URL-encoded as `%26` or Burp interprets it as an HTTP parameter separator.
