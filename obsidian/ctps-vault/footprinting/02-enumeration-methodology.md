# THEORY — Enumeration Methodology

## ID
14

## Module
Footprinting

## Kind
methodology

## Title
Section 2 — Enumeration Methodology

## Description
The 6-layer static methodology for external/internal pentesting — Internet Presence → Gateway → Accessible Services → Processes → Privileges → OS Setup. A framework, not a step-by-step.

## Tags
methodology, layers, framework, recon, pentest

## TL;DR — What's Important
- Methodology is a **framework**, not a checklist. It tells you *what kinds of things to look for*, not the order to run tools.
- 6 layers across 3 levels: **infrastructure-based**, **host-based**, **OS-based**.
- Layers 1–2 (Internet Presence, Gateway) don't apply to internal/AD networks — they have their own layers covered in other modules.
- Goal at every layer: find the **gap**, don't bash through the wall.

## The 6 Layers
| # | Layer | Goal | Information Categories |
|---|-------|------|------------------------|
| 1 | **Internet Presence** | Identify external infrastructure | Domains, Subdomains, vHosts, ASN, Netblocks, IPs, Cloud Instances, Security Measures |
| 2 | **Gateway** | Identify defensive measures | Firewalls, DMZ, IPS/IDS, EDR, Proxies, NAC, Network Segmentation, VPN, Cloudflare |
| 3 | **Accessible Services** | Map exposed interfaces and services | Service Type, Functionality, Configuration, Port, Version, Interface |
| 4 | **Processes** | Identify internal data flows | PID, Processed Data, Tasks, Source, Destination |
| 5 | **Privileges** | Identify what you can do as which user | Groups, Users, Permissions, Restrictions, Environment |
| 6 | **OS Setup** | Inspect internal OS configuration | OS Type, Patch Level, Network config, OS Environment, Configuration files, sensitive private files |

## Three Enumeration Levels
- **Infrastructure-based** — layers 1–2 (Internet Presence, Gateway).
- **Host-based** — layer 3 (Accessible Services). *This module focuses here.*
- **OS-based** — layers 4–6 (Processes, Privileges, OS Setup). Covered in privesc modules.

## Mental Model — The Wall and the Maze
- Each layer is a wall with a gap somewhere. Find the gap; don't smash through, because the spot you smash may not lead anywhere on the next wall.
- The full pentest is a maze: many gaps, but not all lead inside. You won't always find them all in the time available.
- Even after a 4-week test you can't say "no vulns exist." Adversaries who study a target for months will always know more about it than you. SolarWinds is the canonical example.

## Why It Matters
- Without a methodology you'll skip layers (typically 4–6) and miss easy wins.
- It externalizes the order of attack so you don't fall back on personal habits and miss vectors that don't match your comfort zone.

## Methodology vs Cheatsheet
- **Methodology** = systematic approach to investigation (this document).
- **Cheatsheet** = the specific tools/commands per service. Lives separately in module-specific notes.
- Tools change constantly; methodology is stable. Don't conflate them.

## Key Takeaways
- One layer at a time. Don't jump to "Accessible Services" exploitation while still missing assets in "Internet Presence".
- Layers 1–2 are external-only. Internal AD pentests start at a different model.
- Find the gap that leads to the next wall, not just any gap.

## Gotchas
- Treating the layers as a strict checklist makes you brittle. They're a coverage model.
- Confusing methodology with toolset. New tool ≠ new methodology.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[01-enumeration-principles]] | [[03-domain-information]] →
<!-- AUTO-LINKS-END -->
