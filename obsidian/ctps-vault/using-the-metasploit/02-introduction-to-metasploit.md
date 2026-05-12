# THEORY — Introduction to Metasploit

## ID
601

## Module
Using the Metasploit Framework

## Kind
theory

## Title
Section 2 — Introduction to Metasploit

## Description
Overview of the Metasploit Project — Framework vs Pro, what msfconsole brings, and the on-disk layout of the framework so you know where modules, plugins, scripts, and tools live.

## Tags
metasploit, msfconsole, architecture, msf-pro, filesystem-layout

## Commands
- `ls /usr/share/metasploit-framework/modules`
- `ls /usr/share/metasploit-framework/plugins/`
- `ls /usr/share/metasploit-framework/scripts/`
- `ls /usr/share/metasploit-framework/tools/`

## What This Section Covers
The Metasploit Project is a Ruby-based, modular pentest platform with a database of exploit POCs you can launch with a few commands. There are two editions: **Framework** (open-source, free, community-driven) and **Pro** (paid, GUI, task chains, social engineering, vulnerability validation, Nexpose integration).

## Metasploit Pro vs Framework
| Feature | Framework | Pro |
|---------|-----------|-----|
| Cost | Free | Paid |
| Interface | msfconsole CLI | GUI + own console |
| Task Chains | ✗ | ✓ |
| Social Engineering | ✗ | ✓ |
| Vulnerability Validations | ✗ | ✓ |
| Quick Start Wizards | ✗ | ✓ |
| Nexpose Integration | ✗ | ✓ |

## Why msfconsole
- The only supported way to access most of the Framework.
- All-in-one centralized console.
- Most stable MSF interface.
- Full readline, tab completion, command completion.
- Can shell out to external commands without leaving the prompt.

## On-Disk Layout — `/usr/share/metasploit-framework/`
| Folder | What lives there |
|--------|------------------|
| `data/`, `lib/` | Functional internals of msfconsole |
| `documentation/` | Technical project docs |
| `modules/` | `auxiliary  encoders  evasion  exploits  nops  payloads  post` |
| `plugins/` | 3rd-party Ruby plugins (`.rb`) — Nessus, OpenVAS, sqlmap, wmap, etc. |
| `scripts/` | `meterpreter  ps  resource  shell` helper scripts |
| `tools/` | CLI utilities callable from msfconsole — `context  dev  docs  exploit  hardware  memdump  modules  password  payloads  recon` |

## Engagement Structure (preview of the rest of the module)
1. **Enumeration** — Service Validation, Vulnerability Research
2. **Preparation** — Code Auditing
3. **Exploitation** — Module Execution
4. **Privilege Escalation**
5. **Post-Exploitation** — Pivoting, Data Exfiltration

## Key Takeaways
- Know `/usr/share/metasploit-framework/` cold — when you add a custom module or plugin, you'll be dropping files into `modules/` or `plugins/` here.
- Module categories matter for `search` filtering: `auxiliary`, `encoders`, `evasion`, `exploits`, `nops`, `payloads`, `post`.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|using-the-metasploit]]
← [[01-preface]] | [[03-introduction-to-msfconsole]] →
<!-- AUTO-LINKS-END -->
