# Escaping Restricted Shells

## ID
700

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 7 — Escaping Restricted Shells

## Description
Covers restricted shell types (rbash, rksh, rzsh), enumeration of shell restrictions, and multiple escape techniques including command injection, SSH bypasses, and abuse of editors/pagers/languages to break out into an unrestricted shell.

## Tags
restricted-shell, rbash, privilege-escalation, shell-escape, ssh-bypass, post-exploitation

## Commands
- `echo $SHELL`
- `echo $0`
- `ssh <USER>@<TARGET_IP> -t "bash --noprofile"`
- `vi newfile.txt` → `:set shell=/bin/bash` → `:shell`
- `python -c 'import os; os.system("/bin/sh")'`
- `awk 'BEGIN {system("/bin/sh")}'`
- `find / -name <FILE> -exec /bin/bash \;`
- `env` or `printenv`
- `sudo -l`

## What This Section Covers
Restricted shells (rbash, rksh, rzsh) limit what users can execute — changing directories, modifying environment variables, or running arbitrary binaries may all be blocked. Escaping these restrictions is a key post-exploitation step during Linux privilege escalation, since gaining an unrestricted shell is a prerequisite for further enumeration and privesc. The section walks through reconnaissance of the restricted environment and multiple breakout techniques ranging from trivial to advanced.

## Methodology
1. **Identify your shell** — run `echo $SHELL` and `echo $0` to confirm you are in a restricted shell (rbash in ~90% of cases).
2. **Enumerate the environment** — check available commands (try TAB-TAB), review `env`/`printenv` for writable variables, run `sudo -l`, and test which operators work (`|`, `;`, `>`, `>>`, `<`, `$()`, `${}`).
3. **Check for SUID binaries** — any SUID command with escape features (vim, find, nmap, etc.) can be leveraged to spawn a root shell.
4. **Check available languages** — Python, Perl, PHP, Ruby, Lua, Expect — any of these can spawn `/bin/sh`.
5. **Attempt escape** — try the simplest techniques first (command injection, chaining, SSH bypass) before moving to editors, pagers, and language interpreters.

## Multi-step Workflow (optional)
```
# 1. Confirm restricted shell
echo $SHELL
echo $0

# 2. Escape via SSH bypass (noprofile)
ssh htb-user@<TARGET_IP> -t "bash --noprofile"
# Press Ctrl+C if the shell hangs

# 3. Verify unrestricted shell & grab flag
echo $0
ls
cat flag.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Read flag.txt | HTB{35c4p3_7h3_r3stricted_5h311} | SSH back into the box with `ssh htb-user@<IP> -t "bash --noprofile"`, Ctrl+C, then `cat flag.txt` |

## Key Takeaways
- `echo $SHELL` / `echo $0` is always the first move — know what you're trapped in before trying to break out.
- The SSH `bash --noprofile` bypass (`ssh user@host -t "bash --noprofile"`) works because it spawns a new bash session that skips profile scripts that enforce restrictions — trivial but highly effective against rbash.
- Editors (vi/vim, ed, pico), pagers (less, more, man), and language interpreters (python, perl, lua, ruby, php) are the most common escape vectors because they all support `!command` or `system()` calls.
- SUID binaries with escape features (find, nmap interactive, git, zip, tar) are goldmines — they execute as the file owner (often root), turning a shell escape into immediate privilege escalation.
- Defenders should use allow-lists (not deny-lists), restricted editor versions (rvim, red, rnano), and make `$EDITOR`/`$VISUAL` read-only — the number of commands with hidden escape features far exceeds what any admin can enumerate.
- Command chaining operators (`;`, `|`), substitution (`$(...)`, backticks), and injection into allowed command arguments are all viable if the shell doesn't explicitly block them.

## Gotchas
- The restricted shell may hang or behave oddly after the SSH bypass — press **Ctrl+C** to get a clean prompt before running commands.
- `bash --noprofile` is different from `bash --norc`: `--noprofile` skips login profile scripts (where rbash restrictions are typically set), while `--norc` skips `.bashrc`. For escaping rbash, `--noprofile` is the one you want.
- Some restricted shells block `/` in commands — if you can't type `/bin/bash`, try relative paths, command substitution, or spawning a shell through an allowed binary instead.
