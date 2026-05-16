---
id: gs-101
module: Getting Started
kind: note
title: "Infosec Fundamentals & Pentest Role"
description: "CIA triad, Risk Management process, Red Team vs Blue Team, role of penetration testers in identifying organizational risk."
tags: [infosec, cia-triad, risk-management, red-team, blue-team, compliance, business-continuity]
---

# Infosec Fundamentals

## CIA Triad (Core Tenant)

All infosec work protects three properties of data:

1. **Confidentiality** — Data not accessed by unauthorized parties
2. **Integrity** — Data not modified by unauthorized parties
3. **Availability** — Data accessible when needed (not disrupted)

**In pentest context:** Identify where CIA is broken, report to org so they can remediate.

---

## Risk Management Process (5 Steps)

| Step | Task | Example |
|------|------|---------|
| **Identify Risk** | List all risks org faces (legal, environmental, market, regulatory, technical) | "Legacy PHP app running unpatched version with known RCE" |
| **Analyze Risk** | Map risks to policies, business processes; determine impact & probability | "RCE impact: HIGH (remote compromise), Probability: HIGH (unpatched)" |
| **Evaluate/Prioritize** | Rank risks; decide: Accept, Avoid, Control (mitigate), Transfer (insure) | "CONTROL: patch immediately; if can't, add WAF" |
| **Deal with Risk** | Eliminate or contain risk via stakeholder coordination | "Schedule patching window, test in staging, deploy" |
| **Monitor Risk** | Watch for changes (new versions, new vulns, config drift) | "Monitor for new CVEs in that app, re-scan quarterly" |

**Pentest fits here:** We conduct tests to *Identify* and *Analyze* risks, feeding the cycle.

---

## Red Team vs Blue Team

| Aspect | Red Team | Blue Team |
|--------|----------|-----------|
| **Role** | Attacker—break into org, find weaknesses | Defender—strengthen defenses, respond to incidents |
| **Common tasks** | Penetration testing, social engineering, security assessments | Incident response, threat analysis, policy enforcement, security tools management |
| **Org size** | Smaller orgs; external consultants common | Larger orgs; in-house team (SOC) |
| **Career prevalence** | ~20% of infosec jobs | ~80% of infosec jobs |

---

## Role of Penetration Tester

**Goal:** Identify risks in external + internal networks; reproduce vulnerabilities for client; provide remediation guidance.

**Activities:**
- Network penetration testing (find remote access paths)
- Web application testing (identify web vulns: SQLi, XSS, auth bypass)
- Red team assessments (scenario-based: emulate real threat actor)
- Social engineering / phishing assessments (test employee awareness)

**Deliverable:** Report with:
- Vulnerabilities found (with severity: Critical, High, Medium, Low)
- How to reproduce each (proof of concept)
- Remediation steps
- Overall risk posture

---

## Key Takeaway for Exam

Pentesting is **risk identification**, not just finding vulns for fun. On exam, your goal is to:
1. Compromise system thoroughly (find all vectors).
2. Document findings clearly (what, where, impact, fix).
3. Demonstrate you understand business impact (not just "I got root").
