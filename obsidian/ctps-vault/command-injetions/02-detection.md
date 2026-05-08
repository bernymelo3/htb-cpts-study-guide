# Section 2 — Detection

## ID
600

## Module
Command Injections

## Kind
notes

## Title
Section 2 — Detection

## Description
Demonstrates how to detect OS command injection by appending injection operators to user input in a ping-based Host Checker and observing error messages or command output.

## Tags
command-injection, detection, injection-operators, ping, host-checker

## Commands
- 127.0.0.1; <COMMAND>
- 127.0.0.1 | <COMMAND>
- 127.0.0.1 & <COMMAND>
- 127.0.0.1 && <COMMAND>
- 127.0.0.1 || <COMMAND>
- 127.0.0.1 `<COMMAND>`
- 127.0.0.1 $(<COMMAND>)

## What This Section Covers
Command injection detection works by appending OS operators to user-controlled input and checking whether the application's response changes from its normal behavior. If the output differs (error, extra output, or different formatting), the input is likely reaching a system command unsanitized. The target app wraps user input directly into `ping -c 1 <INPUT>`.

## Methodology
1. Confirm baseline behavior by entering a normal IP like `127.0.0.1` and observing standard ping output.
2. Append an injection operator and a test command (e.g., `127.0.0.1; whoami`) to the input field.
3. Compare the response to the baseline — any change (error message, extra output, missing output) indicates the input reaches a shell command.
4. Cycle through all operators (`;`, `|`, `&`, `&&`, `||`, backticks, `$()`) to determine which ones the application accepts or blocks.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Try adding any of the injection operators after the ip. What did the error message say? | **Please match the requested format.** | Entered `127.0.0.1;` in the Host Checker input field — triggered client-side HTML pattern validation |

## Key Takeaways
- Detection and exploitation of basic command injection use the same technique — inject and observe.
- All listed operators work across PHP, .NET, and NodeJS backends on both Linux and Windows, except `;` which fails on Windows CMD (works on PowerShell).
- The underlying command is `ping -c 1 <INPUT>` — knowing the backend command helps craft effective payloads.
- URL-encoded versions of operators (`%3b`, `%0a`, `%26`, `%7c`) are useful when the app filters the raw characters but not their encoded forms.

## Gotchas
- Semicolon `;` does not work on Windows CMD — only on PowerShell and Linux shells.
- `&` in a URL query string has special meaning (parameter separator), so always URL-encode it as `%26` when injecting via GET requests.
- Pipe `|` only shows the second command's output, which can make it look like the first command failed when it actually ran.
