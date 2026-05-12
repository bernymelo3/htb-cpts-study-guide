# NOTE — Anatomy of a Shell

## ID
402

## Module
Shells & Payloads

## Kind
notes

## Title
Section 3 — Anatomy of a Shell

## Description
Separates the three components of a CLI session (OS + terminal emulator + command-language interpreter) and shows how to identify which shell is in use on a system.

## Tags
shell, terminal-emulator, bash, powershell, env, ps

## Commands
- `ps` — list running processes (shows shell binary like `bash`, `pwsh`)
- `env` — print environment variables (look at `SHELL=`)
- `echo $SHELL` — quick shell-binary check
- `pwsh` — launch PowerShell Core on Linux
- `$PSVersionTable` — print PowerShell version info
- `$PSVersionTable.PSEdition` — get edition (Core/Desktop)

## What This Section Covers
A "shell" is not one thing — it's three layered components: the OS, the terminal emulator (xterm, iTerm, Konsole, etc.), and the command-language interpreter (Bash, PowerShell, Zsh, Ksh, POSIX). Identify which is in use because that determines what commands and payloads will work.

## Terminal Emulators by OS
| Emulator | OS |
|----------|----|
| Windows Terminal, cmder, PuTTY | Windows |
| kitty, Alacritty | Cross-platform |
| xterm, GNOME, MATE, Konsole | Linux |
| Terminal, iTerm2 | macOS |

## Identifying the Active Shell
1. **By prompt**: `$` typically = Bash/Ksh/POSIX. `>` or `PS C:\>` = PowerShell. `#` = root.
2. **By process**: `ps` — the active shell binary shows up.
3. **By env**: `env | grep SHELL` — explicit declaration.

### Example: `ps` showing Bash
```shellsession
$ ps
    PID TTY          TIME CMD
   4232 pts/1    00:00:00 bash
  11435 pts/1    00:00:00 ps
```

### Example: `env` showing Bash
```shellsession
$ env
SHELL=/bin/bash
```

## PowerShell Identification (Cross-Platform)
PowerShell Core (`pwsh`) runs on Linux/macOS. Edition differentiates from full PowerShell:
- **Core** — cross-platform, .NET Core based
- **Desktop** — Windows-only, .NET Framework

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Which two shell languages did we experiment with in this section? | **Bash&PowerShell** | Section opens MATE terminal with `$` prompt (Bash), then a blue-icon variant launches PowerShell. |
| Q2 — Edition of PowerShell from `$PSVersionTable` | **Core** | Run `pwsh` then `$PSVersionTable.PSEdition` → returns `Core`. |

## Key Takeaways
- The terminal emulator and the command interpreter are independent — same emulator can host different shells.
- On a popped target, identify the shell *first*; it drives every subsequent command and payload choice.
- `pwsh` on Linux = `Core` edition; full PowerShell on Windows = `Desktop`.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|shells-payloads]]
← [[02-cat5-engagement-prep]] | [[04-bind-shells]] →
<!-- AUTO-LINKS-END -->
