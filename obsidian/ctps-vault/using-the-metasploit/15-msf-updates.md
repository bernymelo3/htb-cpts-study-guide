# THEORY — Metasploit Framework Updates (August 2020 / MSF6)

## ID
614

## Module
Using the Metasploit Framework

## Kind
theory

## Title
Section 15 — Metasploit Framework Updates (August 2020 / MSF6)

## Description
Breaking changes from MSF5 → MSF6 — end-to-end AES on Meterpreter sessions, SMBv3 client, polymorphic Windows shellcode, cleaner DLL artifacts, and the Mimikatz → Kiwi plugin replacement.

## Tags
metasploit, msf6, msf5, aes-encryption, smbv3, polymorphic-shellcode, kiwi, mimikatz, changelog

## TL;DR — What's Important
- **MSF6 sessions are not back-compatible with MSF5.** MSF5-generated payloads will not connect to MSF6 listeners and vice versa.
- **All five Meterpreter implementations** (Windows, Python, Java, Mettle, PHP) now use **AES end-to-end** between attacker and target.
- **SMBv3 client support** added.
- **Polymorphic Windows shellcode generator** shuffles instructions each run to dodge static signatures.
- **Mimikatz plugin → Kiwi.** `load mimikatz` actually loads Kiwi now.

## Why It Matters
If you train on MSF5 box images and try to attack from MSF6 (or vice versa) the handshakes won't complete. Re-generate payloads after upgrading the Framework version.

## Generation Features (MSF6 additions)
- End-to-end encryption across Meterpreter sessions for all five implementations.
- SMBv3 client support → modern Windows targets exploitable without falling back to SMBv1.
- New polymorphic payload generation routine for Windows shellcode → better AV/IDS evasion vs the same-bytes-every-time MSF5 generator.

## Expanded Encryption
- Higher cost for defenders to write signature-based rules against MSF network operations.
- **All** Meterpreter payloads encrypt attacker ↔ target traffic with AES.
- SMBv3 encryption → harder to write signature-based detections for key SMB operations.

## Cleaner Payload Artifacts
- DLLs used by Windows Meterpreter now resolve necessary functions **by ordinal instead of name** → no readable Windows API strings in memory.
- The standard `ReflectiveLoader` export used by reflectively-loadable DLLs is no longer present as text data.
- Commands Meterpreter exposes to the Framework are encoded as **integers instead of strings** → no command-name strings to grep on in memory.

## Plugins
- Old `Mimikatz` Meterpreter extension is **removed**.
- Replaced by **Kiwi** — same capability, modern code.
- For backward compatibility, `load mimikatz` resolves to Kiwi anyway.

## Payloads
The static shellcode generation routine was replaced with a **randomization routine** that shuffles instructions on each generation. This adds polymorphic properties to the critical stub so it doesn't look identical between two `msfvenom` runs of the same payload spec.

## Practical Notes for the Exam
- If a Meterpreter session "dies" immediately on connect, check that attacker and target tooling are on the same major version.
- For credential extraction, reach for Kiwi commands inside Meterpreter — `load kiwi` (or `load mimikatz` aliased) → `creds_all`, `kerberos_ticket_use`, etc. Don't expect MSF5-era Mimikatz docs to map 1:1.
- The polymorphism doesn't fully evade modern AV — see [[14-firewall-ids-ips-evasion]] for what actually works.

## Closing Notes from the Module
- MSF is a powerful framework when used correctly — not a substitute for skill, but excellent for data tracking, post-exploitation, and pivoting.
- Worth using on lab boxes (Dante Pro Lab, retired HTB machines) to internalize the workflow before relying on it in a graded environment.
- If you prefer manual exploitation, that's fine — work with what you're comfortable with.

## Key Takeaways
- Treat MSF version as a hard constraint when generating standalone payloads → regenerate on every Framework upgrade.
- `load mimikatz` is a Kiwi alias post-Aug-2020 — the docs are different now.
- AES + ordinal-resolution + integer-encoded commands raise the bar for in-memory forensics but don't make you invisible.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|using-the-metasploit]]
← [[14-firewall-ids-ips-evasion]] | (last note) →
<!-- AUTO-LINKS-END -->
