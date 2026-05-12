# THEORY — Introduction to MSFconsole

## ID
602

## Module
Using the Metasploit Framework

## Kind
theory

## Title
Section 3 — Introduction to MSFconsole

## Description
Launching msfconsole, keeping modules updated via apt, and the five-stage MSF Engagement Structure used as the spine of the rest of the module.

## Tags
metasploit, msfconsole, update, engagement-structure, enumeration

## Commands
- `msfconsole`
- `msfconsole -q`
- `sudo apt update && sudo apt install metasploit-framework`
- `help`

## What This Section Covers
Start msfconsole, quiet the banner with `-q`, keep it up-to-date via apt (the modern replacement for `msfupdate`), and understand the engagement framework that the module's later sections map to.

## Launching MSFconsole
| Command | Behavior |
|---------|----------|
| `msfconsole` | Launch with full banner |
| `msfconsole -q` | Quiet — skip banner |
| `help` | Show all available commands |

## Keeping Modules Updated
Old method was `msfupdate`. Modern method:
```
sudo apt update && sudo apt install metasploit-framework
```
This pulls in new modules and features pushed to the upstream framework.

## Enumeration Precedes Exploitation
Before picking an exploit you need:
- Which **services** are public-facing on the target?
- Which **versions** are running?

Versions are the key — unpatched versions of previously-vulnerable services or outdated code in publicly-accessible platforms are typically your entry point. Always scan the target's IP first to map service + version.

## MSF Engagement Structure
| Stage | Subcategories |
|-------|---------------|
| 1. Enumeration | Service Validation, Vulnerability Research |
| 2. Preparation | Code Auditing |
| 3. Exploitation | Module Execution |
| 4. Privilege Escalation | (covered in later modules) |
| 5. Post-Exploitation | Pivoting, Data Exfiltration |

This division makes it easier to find the right MSF feature for the phase you're in.

## Key Takeaways
- `-q` is the default professional posture — the banner is for newcomers.
- Treat `apt install metasploit-framework` as the update path; `msfupdate` is legacy.
- Map the work you're doing back to the 5-stage structure — it tells you which module category to reach for.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|using-the-metasploit]]
← [[02-introduction-to-metasploit]] | [[04-modules]] →
<!-- AUTO-LINKS-END -->
