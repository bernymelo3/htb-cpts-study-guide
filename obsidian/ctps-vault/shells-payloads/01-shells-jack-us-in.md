# THEORY — Shells Jack Us In, Payloads Deliver Us Shells

## ID
400

## Module
Shells & Payloads

## Kind
theory

## Title
Section 1 — Shells Jack Us In, Payloads Deliver Us Shells

## Description
Defines what a "shell" means in pentesting and why establishing one is the goal after vulnerability exploitation — interactive OS access enables enumeration, privesc, pivoting, and persistence.

## Tags
shells, payloads, fundamentals, theory, cli, terminology

## TL;DR — What's Important
- A **shell** = interface to OS commands + filesystem (Bash, Zsh, cmd, PowerShell).
- "Caught a shell" / "popped a shell" = exploited a vuln → got interactive remote access.
- CLI shells are stealthier than graphical (VNC/RDP) and easier to automate.
- A **payload** is the code/command crafted to exploit a vuln and deliver that shell.

## What This Section Covers
The first lesson frames the whole module: every exploitation milestone ends with "do we have a shell?" The shell gives direct OS access — without it, we can't enumerate further, escalate, pivot, or persist.

## Shell Perspectives
| Perspective | Description |
|-------------|-------------|
| **Computing** | Text-based userland environment (Bash, Zsh, cmd, PowerShell). |
| **Exploitation** | Result of exploiting a vuln or bypassing security to gain interactive host access (e.g. EternalBlue → cmd-prompt remote). |
| **Web** | Web shell — exploits an upload vuln to issue commands via the browser. |

## Payload — Definitions Across Contexts
| Context | Meaning |
|---------|---------|
| Networking | Data portion of a packet (headers stripped). |
| Computing | Action part of an instruction set. |
| Programming | Data carried by a language instruction. |
| **Exploitation** | Code crafted to exploit a vulnerability — includes malware/ransomware. |

## Why Shells Matter (Beyond Just "Access")
- Enumerate the host for privesc/pivot vectors
- Maintain persistence
- Easier to use attack tools, exfil data
- CLI is harder to detect than graphical sessions
- Faster + scriptable

## Key Takeaways
- Shells are the universal goal — every chapter that follows is about *how* to get one or *what to do* once you have it.
- "Payload" and "malware" are the same idea — payload is the neutral term.
- Prefer CLI access over GUI for stealth and automation.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|shells-payloads]]
← _start_ | [[02-cat5-engagement-prep]] →
<!-- AUTO-LINKS-END -->
