## ID
601

## Module
Documentation & Reporting

## Kind
notes

## Title
Section 3 — Types of Reports

## Description
Defines every assessment and report type a pentester may be contracted for, covering vulnerability assessments, penetration test variants (black/grey/white box, evasive), inter-disciplinary assessments, and the full report lifecycle from draft to attestation.

## Tags
documentation, reporting, assessment-types, penetration-testing, methodology

## Commands
<!-- No commands in this section — theory only -->

## What This Section Covers
Covers the full landscape of assessment types (vulnerability assessment, internal/external pentest, purple team, cloud, IoT, web app, hardware) and maps each to its corresponding report type (draft, final, post-remediation, attestation, slide deck, findings spreadsheet, vulnerability notification). Understanding these distinctions is critical for scoping engagements correctly and producing the right deliverable for each client.

## Assessment Types Reference

### Vulnerability Assessment
- Automated scan only — **no exploitation attempted**
- Can be authenticated or unauthenticated
- Goal: enumerate vulnerabilities and validate scanner results (real vs false positive)
- Report focuses on themes, severity counts, and procedural deficiencies

### Penetration Testing
| Perspective | Description |
|---|---|
| **Black Box** | No info given — just the company name (external) or a network connection (internal) |
| **Grey Box** | Given in-scope IPs/CIDR ranges only |
| **White Box** | Full access — credentials, source code, configs |

| Evasion Level | Description |
|---|---|
| **Non-evasive** | No stealth — find as many vulns as possible |
| **Hybrid evasive** | Start stealthy, get noisier — tests detection thresholds |
| **Full evasive** | Remain undetected as long as possible — simulates advanced attacker |
| **Adversary simulation** | Long-term (months), few staff aware, tests full incident response |

### Internal vs External
| Type | Perspective |
|---|---|
| **External** | Anonymous attacker on the internet targeting public-facing systems; uses OSINT |
| **Internal** | Behind the firewall — anonymous or authenticated user; focuses on AD compromise, lateral movement, privilege escalation |

### Inter-Disciplinary Assessments
| Type | Key Detail |
|---|---|
| **Purple Team** | Pentester (red) + incident responder (blue) working together; validates alerting/detection rules |
| **Cloud Focused** | Requires cloud architecture knowledge; containers/serverless need specialized methodology |
| **IoT** | Three components: network, cloud, application — plus optional hardware layer |
| **Web App** | May combine app testing (authenticated/role-based) + network pentesting beyond the app |
| **Hardware** | Physical device security; establish RoE on destructive testing before starting |

## Report Lifecycle

### 1. Draft Report
- Submit first — give client time to review independently
- Client may add management responses, tweak language, or restructure
- Schedule a review call to answer questions and incorporate feedback

### 2. Final Report
- Issued after client confirms satisfaction with the draft
- Required for compliance — auditing firms typically won't accept a draft

### 3. Post-Remediation Report
- Retest **only** the original findings on **only** the originally affected hosts
- Set a time limit — retesting months later causes scope creep and apples-to-oranges comparisons
- Do **not** run new large-scale scans — new findings will spiral the scope out of control
- If client pushes for scope expansion or severity modifications: hold your ground but offer alternatives (e.g., documented remediation plan with timeline that auditors may accept)

### 4. Attestation Report
- 1–2 pages only — suitable to share with third-party vendors/auditors
- Contains: number of findings, approach taken, general environment comments
- **No** technical details, credentials, or sensitive specifics

### 5. Other Deliverables
| Deliverable | Purpose |
|---|---|
| **Slide Deck** | Executive or technical audience — use anecdotes and current events, not just graphs |
| **Findings Spreadsheet** | All finding fields in tabular format; use pivot tables for severity/category analytics |
| **Vulnerability Notification** | Out-of-band alert for critical findings found mid-assessment |

### Vulnerability Notification — When to Issue
Issue immediately (don't wait for the report) for:
- Internet-exposed, directly exploitable finding → unauthenticated RCE or sensitive data exposure
- Weak/default credentials leading to the same
- Whatever threshold is agreed upon at project kickoff (some clients want all highs/criticals, even internal)

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Inlanefreight wants a mostly automated assessment with no exploitation attempted. What type? | Vulnerability Assessment | Defined in the Vulnerability Assessment section |
| Nicolas is given only the company name and a network connection — what perspective is he testing from? | Black Box | Black box = no info beyond company name / network connection |

## Key Takeaways
- **Vulnerability assessment ≠ penetration test** — no exploitation is attempted in a VA; knowing this distinction matters when scoping and pricing engagements.
- **Black box = zero info** — not just "limited info." If you're given IPs, that's grey box.
- **Post-remediation testing has hard boundaries** — only retest original findings on original hosts within a reasonable timeframe; scope creep here is a major professional hazard.
- **Draft → Final is not bureaucracy** — auditors won't accept a draft for compliance purposes; always issue a separate final.
- **Attestation reports exist for a reason** — clients should never hand full technical pentest reports to third-party vendors; strip it down to a 1–2 page attestation.
- **Vulnerability notifications are time-sensitive** — a critical RCE sitting on the internet can't wait until the report is done; establish the threshold at kickoff so the client isn't surprised.

## Gotchas
- If a client asks you to retest months later and the environment has changed significantly, treat it as a new engagement — trying to retrofit it as a post-remediation test will compromise the integrity of your findings.
- If a client pressures you to soften severity ratings or remove findings to satisfy an auditor, hold firm — offer the documented remediation plan route instead of compromising the report.
- Hybrid evasive assessments: once detected, the client typically asks you to switch to non-evasive for the remainder — agree to this at kickoff so there's no ambiguity mid-engagement.
