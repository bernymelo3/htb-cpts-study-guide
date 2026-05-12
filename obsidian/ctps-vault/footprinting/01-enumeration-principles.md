# THEORY — Enumeration Principles

## ID
13

## Module
Footprinting

## Kind
theory

## Title
Section 1 — Enumeration Principles

## Description
Mindset framing: enumeration is information gathering (active + passive), distinct from OSINT (passive only). The goal is not to break in but to find every way in.

## Tags
enumeration, principles, mindset, methodology, theory

## TL;DR — What's Important
- **Enumeration ≠ OSINT.** OSINT is purely passive third-party recon; enumeration includes active scans and is iterative.
- **Don't kick the door — map the building first.** Brute-forcing SSH/RDP/WinRM with weak creds is loud and gets you blacklisted before you've understood the target.
- **Goal:** find *all* the ways in, not just the obvious one.
- **Pay attention to what you don't see.** Absence of a service/port/banner is information too.
- **Treasure-hunter analogy:** plan, gather gear, study maps. Don't dig random holes.

## Concept Overview
Enumeration is a loop — every piece of data you find feeds the next round of questions. It pulls from domains, IPs, services, banners, certificates, DNS records, employee data, and anything else externally observable. The mistake most testers make is fixating on what's visible (open ports, login pages) instead of building a model of how the company actually functions.

## The Three Principles
| # | Principle |
|---|-----------|
| 1 | There is more than meets the eye. Consider all points of view. |
| 2 | Distinguish between what we see and what we do not see. |
| 3 | There are always ways to gain more information. Understand the target. |

## The Nine Questions
Keep these visible while working:
1. What can we see?
2. What reasons can we have for seeing it?
3. What image does what we see create for us?
4. What do we gain from it?
5. How can we use it?
6. What can we **not** see?
7. What reasons can there be that we do not see?
8. What image results for us from what we do not see?
9. (apply principle 3 — push for more)

## Why It Matters
- The core skill being tested in CPTS is technical understanding, not exploitation muscle memory. When you don't know what to do next, the gap is usually understanding the protocol — not skill at running tools.
- "Brute force first" is the canonical wrong answer: noisy, gets blacklisted, blocks further testing, and tells you nothing about how the company actually works.

## Key Takeaways
- Enumeration is iterative — every finding triggers new queries.
- The principles don't change; only the exceptions do.
- Write down the principles + 9 questions somewhere you can see them during an engagement.

## Gotchas
- Confusing OSINT with enumeration leads to under-scoping the active phase.
- Skipping enumeration to "just try exploits" wastes contracted time and burns stealth.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
[[02-enumeration-methodology]] →
<!-- AUTO-LINKS-END -->
