# THEORY — Targets

## ID
604

## Module
Using the Metasploit Framework

## Kind
theory

## Title
Section 5 — Targets

## Description
What "target" means inside an exploit module — the OS/version/arch identifier that supplies the right return address. Includes the `show targets` / `set target <id>` pattern using MS12-063 IE Use-After-Free as the example.

## Tags
metasploit, targets, return-address, msfpescan, exploit-modules

## Commands
- `show targets`
- `info`
- `set target <index>`

## What This Section Covers
A "target" inside an exploit is a per-version identifier (OS + service pack + browser version, etc.) that maps to a specific return address or memory layout. Most exploit modules support multiple targets; you choose one with `set target <id>`. `Automatic` (target `0`) lets MSF do service detection first.

## Workflow Inside a Module
| Command | Use |
|---------|-----|
| `info` | Audit module: description, targets, refs, side effects |
| `show targets` | List supported OS / version IDs |
| `set target <id>` | Pin one explicitly (skip `Automatic` detection) |

`show targets` outside a module errors with `[-] No exploit module selected.`

## Example — MS12-063 IE execCommand UAF
```
exploit/windows/browser/ie_execcommand_uaf

Available targets:
  0  Automatic
  1  IE 7 on Windows XP SP3
  2  IE 8 on Windows XP SP3
  3  IE 7 on Windows Vista
  4  IE 8 on Windows Vista
  5  IE 8 on Windows 7
  6  IE 9 on Windows 7
```
If you know the target's IE + Windows version, pick the matching id:
```
set target 6     # IE 9 on Windows 7
```

## Why Multiple Targets Exist
Targets differ by:
- Service pack
- OS version
- Language pack (changes addresses)
- Software version
- Whether hooks shift addresses

The "return address" can be a `jmp esp`, a jump to a register that identifies the target, or a `pop/pop/ret`. The address depends entirely on what's loaded in the target's memory.

## Identifying a Custom Target
For exploits the module doesn't ship a target for:
1. Get a copy of the target binaries.
2. Use `msfpescan` to locate a suitable return address.

(Both topics — return addresses and PE-scanning — are covered in the Stack-Based Buffer Overflows on Windows x86 module.)

## Key Takeaways
- `info` is the audit step — read it before launching anything, especially against a customer's box.
- `Automatic` (target 0) calls service detection on the target, which adds noise / network traffic. If you already know the version, pin it with `set target <id>` for stealth and reliability.
- Pre-built targets in MSF are not exhaustive — niche service packs, language packs, or hooked processes can require building your own return address with `msfpescan`.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|using-the-metasploit]]
← [[04-modules]] | [[06-payloads]] →
<!-- AUTO-LINKS-END -->
