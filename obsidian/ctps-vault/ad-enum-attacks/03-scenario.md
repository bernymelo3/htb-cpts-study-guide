# THEORY — Scenario

## ID
702

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 3 — Scenario

## Description
Overview of the simulated engagement against Inlanefreight: scope (domains, subnets), methods (passive external enumeration, internal testing, password cracking), and the module’s final skills assessments.

## Tags
scenario, scope, rules-of-engagement, theory, engagement

## TL;DR — What's Important
- **Target domains** – `INLANEFREIGHT.LOCAL` (main), `LOGISTICS.INLANEFREIGHT.LOCAL` (subdomain), `FREIGHTLOGISTICS.LOCAL` (subsidiary with external trust).
- **Internal subnet** – `172.16.5.0/23` (in scope).
- **No phishing or social engineering** – only technical attacks.
- **Two final skills assessments** – one starting from an external breach position, another from an internal attack box.
- **Goal** – obtain domain user credentials, enumerate, gain foothold, move laterally/vertically to compromise all in‑scope domains.

## Concept Overview
This section defines the engagement scenario for the module. You play a penetration tester from CAT‑5 Security tasked with assessing Inlanefreight’s Active Directory environment. The scope includes three domains (parent, subdomain, and a subsidiary with an external forest trust) and the internal subnet `172.16.5.0/23`. External passive enumeration (no active scanning against the real `inlanefreight.com`) is allowed, as is full internal testing starting from an anonymous insider position. Password files may be cracked offline. The module culminates in two skills assessments that require chaining the techniques taught.

## Key Concepts

### In‑Scope Assets
| Range/Domain | Description |
|--------------|-------------|
| `INLANEFREIGHT.LOCAL` | Primary domain (AD + web services) |
| `LOGISTICS.INLANEFREIGHT.LOCAL` | Subdomain |
| `FREIGHTLOGISTICS.LOCAL` | Subsidiary, external forest trust with parent |
| `172.16.5.0/23` | Internal subnet |

### Authorized Methods
- **External information gathering (passive only)** – no active scanning against the real `www.inlanefreight.com`.
- **Internal testing** – start from an anonymous position on the internal network (no credentials provided).
- **Password testing** – offline cracking of captured password files.
- **No social engineering / phishing** – purely technical attacks.

### Goals
1. Obtain domain user credentials.
2. Enumerate the internal domain(s).
3. Gain a foothold.
4. Move laterally and vertically to compromise all in‑scope domains.

## Why It Matters
Understanding the scope and rules of engagement (RoE) is critical in real penetration tests. This scenario mimics a real engagement where the client defines what systems are in scope, what techniques are allowed, and what the ultimate objectives are. As a tester, you must work within these boundaries while still achieving domain compromise. This module’s scenario also sets up the context for all subsequent sections: you will enumerate, find footholds, spray passwords, abuse Kerberos, ACLs, trusts, etc., all aimed at completing the tasks in the final skills assessments.

## Key Takeaways
- **Always document the scope** – IP ranges, domains, subnets, and exclusions. Staying in scope avoids legal or contractual issues.
- **Know the rules of engagement** – are you allowed to phish, use social engineering, or perform DoS? Here, only technical attacks are allowed.
- **External vs internal testing** – the module starts with passive external info gathering (OSINT), then moves to internal enumeration with an anonymous network position.
- **The final assessment has two parts** – one mimics external breach → internal pivot; the other starts from an internal attack host (as many clients request). Complete both to master AD attacks.
- **The subsidiary domain (`FREIGHTLOGISTICS.LOCAL`) has an external forest trust** – trusts are often misconfigured and can be abused to hop from one domain to another.

## Gotchas
- **Passive enumeration only** against the real `inlanefreight.com` – no active scanning or attacks. The module provides a simulated environment, not the real company.
- **The internal subnet is `/23`** – that spans `172.16.4.0` to `172.16.5.255`. Don’t miss hosts in the lower half.
- **No credentials are provided at the start** – you begin from an “anonymous insider” position. All credentials must be discovered or cracked.
- **Offline password cracking is allowed** – you can take captured hashes (Kerberoasting, Responder, etc.) and crack them on your own machine using Hashcat or John.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[02-tools-of-the-trade]] | [[04-external-recon]] →
<!-- AUTO-LINKS-END -->
