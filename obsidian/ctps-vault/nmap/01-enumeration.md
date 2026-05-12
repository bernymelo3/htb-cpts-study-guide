# THEORY — Enumeration

## ID
1

## Module
Nmap

## Kind
notes

## Title
Section 1 — Enumeration

## Description
Sets the mindset for the whole module: enumeration is *the* phase of a pentest. The tools only matter once you know how the underlying services and protocols actually work.

## Tags
enumeration, theory, methodology, mindset, recon

## TL;DR — What's Important
- **Enumeration is collecting maximum information.** More info → easier to find an attack vector.
- **Tools are not knowledge.** Scanners can miss anything they don't have a probe for; service-level understanding fills the gap.
- **Most access comes from misconfigurations or neglect** (default creds, unfiltered ports, exposed banners) — not from zero-days.
- **Manual interaction beats blind scanning** when timeouts, IDS, or odd behaviour skews tool output. A port marked `closed` because Nmap timed out is *not* closed.
- **Two access vectors to look for:** (1) functions/resources that let you interact with the target; (2) information that points you to *more* information.

## Concept Overview
Enumeration is the most critical phase in a pentest. The goal is not to "get in" yet — it is to identify *every* way you could potentially get in. Tools simplify the work, but they do not replace knowledge of how each protocol behaves. Most successful exploitation paths come from configuration gaps the admin overlooked, and those gaps only surface when you understand the service's expected vs. actual behaviour.

## Why It Matters
- Skipping enumeration is the #1 reason students get stuck on a box.
- Tools have timeouts. If a probe doesn't respond fast enough, ports get marked `closed` / `filtered` / `unknown` — and the *one* port you needed disappears from your output.
- Understanding the service (FTP, SMB, SNMP, MSSQL, etc.) lets you craft manual probes that bypass the limits of automated tools.

## Key Takeaways
- Enumeration ≠ "run more scans." It means *knowing what to do* with the output.
- A few hours studying the protocol saves days on exploitation.
- Always cross-check automated tool results with manual interaction (`nc`, `openssl s_client`, `curl`, protocol-specific clients).
- "Closed" in Nmap output may mean dropped/timed out, not actually closed.

## Gotchas
- Default scan timeouts hide ports on slow networks — don't trust a single quick scan.
- Tools alone never substitute for reading the banner or trying the service yourself.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|nmap]]
[[02-introduction-to-nmap]] →
<!-- AUTO-LINKS-END -->
