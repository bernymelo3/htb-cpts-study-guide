# THEORY — Introduction to Active Directory Enumeration & Attacks

## ID
700

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 1 — Introduction to Active Directory Enumeration & Attacks

## Description
Overview of Active Directory (AD), why it's a critical target for attackers, real‑world attack scenarios, and what this module will cover — including enumeration, Kerberoasting, password spraying, and BloodHound.

## Tags
active-directory, enumeration, theory, introduction, ad-attacks

## TL;DR — What's Important
- **AD is everywhere** – ~43% of enterprises use Microsoft AD for identity management. It’s a massive attack surface.
- **Misconfigurations are common** – AD’s complexity means simple permission mistakes, weak passwords, and legacy protocols are routinely exploitable.
- **You need both Windows and Linux skills** – some attacks are easier from Linux (Kerbrute, Impacket), others from Windows (Rubeus, PowerView).
- **Living off the land (LOTL)** – often you must use built‑in Windows tools (net, dsquery, PowerShell) when custom tools are blocked.
- **Real AD attacks chain multiple techniques** – password spray → BloodHound → Kerberoast → pass‑the‑ticket → DCSync → domain admin.

## Concept Overview
Active Directory (AD) is Microsoft’s directory service for Windows domains, providing centralized authentication, authorization, and accounting. Because it controls access to users, computers, groups, and resources, a compromised AD domain often means full network compromise. This module teaches how to enumerate AD from both Windows and Linux attack hosts, then abuse common misconfigurations: password spraying, Kerberoasting, ACL abuse, trusts, and more. You’ll practice with tools like BloodHound, Kerbrute, Rubeus, Impacket, and also native Windows commands.

## Key Concepts

### What is Active Directory?
- Hierarchical database of objects: users, computers, groups, OUs, GPOs, etc.
- Uses LDAP (port 389), Kerberos (port 88), SMB (port 445), DNS (port 53).
- Domain Controllers (DCs) host AD and authenticate logins.

### Why Attack AD?
- One compromised domain admin = entire domain owned.
- Lateral movement across thousands of hosts is possible.
- Trusts between domains expand the attack surface.

### Real‑World Attack Scenarios (from the module)
1. **Kerberoasting + Responder** – crack a TGS ticket, drop SCF files, capture a domain admin’s NetNTLMv2 hash.
2. **Password spraying** – use a NULL session to get user list and policy, spray smartly, then pass‑the‑ticket.
3. **Username enumeration + targeted spraying** – generate user lists from LinkedIn, enumerate valid users with Kerbrute, then spray one common password.

### Tools You Will Use
| Tool | Purpose |
|------|---------|
| BloodHound | AD privilege path mapping |
| Kerbrute | Username enumeration / password spraying (Kerberos pre‑auth) |
| Rubeus | Kerberos attacks from Windows |
| Impacket | Python toolkit (secretsdump, GetNPUsers, etc.) |
| PowerView | AD enumeration from PowerShell |
| Responder | NetNTLM hash capture |
| enum4linux | SMB enumeration from Linux |

## Why It Matters
AD is the crown jewels of most Windows enterprise networks. Gaining a foothold in AD, even as a low‑privileged user, often leads to complete domain compromise due to misconfigurations, over‑privileged accounts, and insecure legacy protocols. As a penetration tester, you must be able to enumerate AD effectively, find attack paths, and chain together techniques to achieve the engagement goals (whether that’s a specific host, data, or full domain admin).

## Key Takeaways
- **Enumerate everything** – users, groups, SPNs, ACLs, trusts, GPOs, sessions. BloodHound automates a lot, but manual checks often find hidden paths.
- **Start wide, then narrow** – first discover attack surface (users, computers), then focus on high‑value targets (domain admins, DCs, sensitive groups).
- **Both Windows and Linux matter** – sometimes you land on a Windows host (use PowerView, Rubeus), sometimes on Linux (use Impacket, Kerbrute). Be bilingual.
- **Don’t lock out accounts** – always check password policy before spraying. Use tools that respect lockout thresholds.
- **Iterate** – an initial low‑privileged user often leads to domain admin after chaining multiple techniques (e.g., enumerate local admin → steal session → DCSync).

## Gotchas
- **Domain controllers may respawn** – in the lab, if you destroy or reset a DC, your session and tools may break. Re‑establish access.
- **Some tools require specific privileges** – Rubeus’s `kerberoast` requires a domain user, `triage` requires admin. Know the difference.
- **Time sync matters** – Kerberos attacks can fail if your attack host’s clock is more than 5 minutes off from the DC.
- **PowerShell execution policy** – may need `Set-ExecutionPolicy Bypass -Scope Process` to load PowerView.
- **Living off the land** – on a locked‑down host, you can’t upload tools. Know `net`, `dsquery`, `nltest`, `reg query`, `wmic`, etc.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
[[02-tools-of-the-trade]] →
<!-- AUTO-LINKS-END -->
