# REFERENCE — Spawning Interactive Shells

## ID
408

## Module
Shells & Payloads

## Kind
reference

## Title
Section 11 — Spawning Interactive Shells

## Description
Cheat-sheet of fallback methods to upgrade a non-tty/jail shell into an interactive shell when Python isn't available. Covers Perl, Ruby, Lua, AWK, find, vim, and `/bin/sh -i`.

## Tags
tty, jail-shell, shell-upgrade, perl, ruby, lua, awk, find, vim, sudo

## Commands
- `/bin/sh -i` — interactive sh
- `perl -e 'exec "/bin/sh";'` — Perl spawn
- `ruby -e 'exec "/bin/sh"'` — Ruby spawn (run inside ruby script)
- `lua -e 'os.execute("/bin/sh")'` — Lua spawn
- `awk 'BEGIN {system("/bin/sh")}'` — AWK
- `find / -name nameoffile -exec /bin/awk 'BEGIN {system("/bin/sh")}' \;` — find + awk
- `find . -exec /bin/sh \; -quit` — find direct
- `vim -c ':!/bin/sh'` — vim from CLI
- `:set shell=/bin/sh` then `:shell` — vim escape
- `ls -la <path>` — file perms check
- `sudo -l` — list sudo permissions

## TL;DR — What's Important
- Python is the primary upgrade vector — but won't always be there.
- Most language runtimes can spawn a shell: Perl, Ruby, Lua, AWK.
- Generic Unix tools (find, vim) also have execution primitives.
- Always check `sudo -l` after upgrading — it needs a stable shell to return anything.

## What This Section Covers
Non-interactive (jail) shells block `su`, `sudo`, prompt-driven commands. Section 10 used `python -c 'import pty; pty.spawn("/bin/sh")'`. This section catalogs the alternatives when Python isn't installed.

## Upgrade Methods by Language/Tool

### `/bin/sh -i`
Crudest: just invoke the shell with `-i`:
```shell
/bin/sh -i
```
Often returns `sh: no job control in this shell` but works.

### Perl
```shell
perl -e 'exec "/bin/sh";'
```

### Ruby
```shell
ruby -e 'exec "/bin/sh"'
```

### Lua
```shell
lua -e 'os.execute("/bin/sh")'
```

### AWK
```shell
awk 'BEGIN {system("/bin/sh")}'
```

### find — Two Variants
**Indirect (via awk):**
```shell
find / -name nameoffile -exec /bin/awk 'BEGIN {system("/bin/sh")}' \;
```
**Direct:**
```shell
find . -exec /bin/sh \; -quit
```
If the named file doesn't exist, the indirect form silently does nothing — pick a file that *does* exist (e.g. `/etc/passwd`).

### Vim
From the command line:
```shell
vim -c ':!/bin/sh'
```
From inside vim:
```
:set shell=/bin/sh
:shell
```

## Substituting the Interpreter
Wherever a snippet uses `/bin/sh` or `/bin/bash`, swap in whatever's actually on the box (`/bin/dash`, `/bin/ash`, `/usr/bin/zsh`). Most Linux boxes have at least `sh` and `bash`.

## Permission Sanity Checks After Upgrade
```shell
ls -la <path/to/file-or-binary>   # file ownership + perms
sudo -l                            # what can this user run as root?
```
`sudo -l` requires a real interactive shell — if you get no output, you're still in the jail.

## Key Takeaways
- Try the cheapest upgrade first: `/bin/sh -i` or `python -c '...pty.spawn...'`.
- Walk down the list; one of Perl/Ruby/Lua/AWK is almost always installed.
- `sudo -l` is the immediate privesc enumeration step *after* you have a real shell — not before.
- Permissions inspection (`ls -la`, `sudo -l`) seeds the next privesc steps.

## Gotchas
- The Ruby and Perl one-liners labeled `(should be run from a script)` in the source — for actual one-liner usage, prefer `perl -e` / `ruby -e` form, not the bare `perl: exec ...` snippet.
- Some restricted shells filter these binaries explicitly — check `echo $PATH` and `which <tool>`.
- AWK isn't installed on every minimal Alpine/BusyBox image (only `busybox awk`); behavior may differ.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|shells-payloads]]
← [[10-infiltrating-linux]] | [[12-introduction-to-web-shells]] →
<!-- AUTO-LINKS-END -->
