# NOTE template — hands-on section (has commands)

---

## ID
531

## Module
Documentation & Reporting

## Kind
lab

## Title
Section 5 — How to Write Up a Finding

## Description
Deep-dive into the anatomy of a professional pentest finding: required components, defensible evidence, actionable remediation, reference quality, and hands-on practice with the WriteHat reporting tool.

## Tags
reporting, findings, documentation, remediation, writehat, evidence, cvss

---

## Commands
<!-- WriteHat is GUI-based via RDP — no CLI commands for the core section.
     Commands below are for the optional exercise (evidence gathering). -->
- xfreerdp /u:htb-student /p:'HTB_@cademy_stdnt!' /v:<TARGET_IP>
- ifconfig
- ipconfig
- sudo responder -I <INTERFACE> -rdwv
- hashcat -m 5600 <HASH_FILE> <WORDLIST>

---

## What This Section Covers

This section teaches how to write a complete, professional, and defensible penetration test finding. It covers the mandatory components every finding must include, how to present reproduction steps clearly for a non-pentest-specialist audience, what makes remediation advice actionable vs. useless, and how to choose high-quality references. The section closes with hands-on practice in WriteHat, a real-world reporting tool used in the field.

---

## Core Concept — Why Finding Quality Matters

The Findings section is the **core deliverable** of any pentest report. A technically brilliant engagement means nothing if the findings are:
- Too vague to reproduce
- Remediations too generic to act on
- Evidence that can be argued or dismissed
- Written for an audience that doesn't exist

The goal: a technical team with no pentest background should be able to **reproduce the issue, understand the risk, and fix it** using only your report.

---

## Anatomy of a Finding — Required Fields

Every finding must include **all six** of these fields, customised to the specific client environment:

| Field | What to Include |
|---|---|
| **Description** | What the vulnerability is, what platform(s) it affects, and the root cause. Don't be generic — reference the specific service, host, or configuration you found. |
| **Impact** | What happens if this is left unresolved. Be concrete: domain compromise, data exfiltration, lateral movement, etc. |
| **Affected Systems** | List specific hosts, IPs, domains, or applications. "The entire domain" is valid if true. |
| **Remediation** | Step-by-step, actionable guidance. Not "fix your registry" — give the exact hive path, the value to change, and a caution note about testing first. |
| **References** | 1–3 external links for further reading. Must be accessible, vendor-agnostic where possible, and not behind a paywall. |
| **Reproduction Steps** | Numbered, figure-by-figure walkthrough of how you found and exploited it, with narrative between each step. |

### Optional / Bonus Fields
- CVE identifier (if applicable)
- OWASP / MITRE ATT&CK IDs
- CVSS score (v3.1 preferred)
- Ease of exploitation / attack probability
- Alternate tools that can validate the finding

---

## Showing Reproduction Steps — Rules

This is where most junior pentesters fail. The reader may have never seen Metasploit, Responder, or Hashcat output in their life.

### Rules for figures / screenshots:

1. **One step per figure.** Never cram multiple actions into one screenshot.
2. **Show full config before running.** If using Metasploit, capture the `show options` output so the reader sees the exact module configuration before you fire it.
3. **Write a narrative between every figure.** Explain what you did, why, and what you're looking at. Captions alone are not enough.
4. **Offer alternative tools** after your primary PoC — just name the tool and link it. Don't re-demonstrate the exploit twice.
5. **Think about copy-paste.** If the client needs to reproduce a web payload, a screenshot of Burp is useless — provide the raw request or payload as text they can copy.

### Defensibility rules — evidence must be airtight:

| Scenario | Insufficient Evidence | Sufficient Evidence |
|---|---|---|
| Cleartext credentials via Basic Auth | Screenshot of the login popup | Login popup WITH fake creds entered + Wireshark showing the cleartext HTTP request |
| RDP / GUI vulnerability | Screenshot of the vulnerable UI | Screenshot showing the target's IP in `ipconfig` output or URL bar on the same screen |
| Web app finding | Screenshot of Burp Suite | Raw HTTP request with payload, copy-pasteable by the client |

> **Rule of thumb:** If a lawyer could argue your screenshot proves nothing, redo the evidence.

### Browser hygiene for screenshots:
- Turn off the bookmarks bar
- Disable personal / unprofessional extensions
- Dedicate a clean browser profile to testing

---

## Effective Remediation — Good vs. Bad

### Principle: be specific enough that a sysadmin can act on it today.

#### Example 1 — Registry hardening

| | Example |
|---|---|
| ❌ Bad | "Reconfigure your registry settings to harden against X." |
| ✅ Good | "To remediate this finding, update the following registry hives. Test in a small group before wide deployment: `HKLM\SYSTEM\CurrentControlSet\...` → Change value `X` to `Y`." |

**Why it matters:**
- The good version saves the client hours of research.
- It builds confidence in you as the assessor.
- It shows you have their best interests in mind.
- It includes a **risk caveat** (registry changes can break things) — critical, because some clients will blindly execute your instructions.

#### Example 2 — Commercial tool recommendations

| | Example |
|---|---|
| ❌ Bad | "Implement [expensive commercial tool] to address this finding." |
| ✅ Good | "The vendor has published a workaround linked below. Commercial tools also exist but may be cost-prohibitive — the workaround provides interim protection until an official patch is released." |

**Why it matters:**
- Not every client has a six-figure security budget.
- Always provide at least one free or vendor-supported path to remediation.
- Acknowledge cost constraints without dismissing tools entirely.

---

## Choosing Quality References

Each finding should link to 1–3 external references. Use this checklist:

| Criteria | Notes |
|---|---|
| ✅ Vendor-agnostic | Don't send clients to a competitor's blog or a vendor trying to sell their product |
| ✅ Thorough walkthrough | Should explain the vuln and remediation, not just mention it |
| ✅ Accessible | No paywalls, no partial articles |
| ✅ Gets to the point | Nobody wants to read 10 pages of preamble |
| ✅ Clean site | No excessive ads, no crypto miner vibes |
| ⭐ Preferred | Write your own blog post — builds your brand, keeps clients on your content |

**Good sources:** MITRE ATT&CK, NIST NVD, Microsoft Security Docs, vendor advisories (for vendor-specific issues), PortSwigger Web Security Academy, HackTricks (for technique context).

---

## Real-World Finding Examples

### ✅ Good Finding — Weak Kerberos Authentication ("Kerberoasting")

| Field | Content |
|---|---|
| **Risk** | High — CVSS 9.5 |
| **Description** | Service accounts in `INLANEFREIGHT.LOCAL` are configured with weak RC4 encryption for Kerberos tickets, allowing any authenticated user to request and offline-crack their service ticket hashes. |
| **Impact** | An attacker with any domain foothold can escalate to the service account's privileges, potentially reaching Domain Admin. |
| **Affected Systems** | Domain: `INLANEFREIGHT.LOCAL` — all SPNs using RC4 |
| **Remediation** | Enable AES256 encryption for Kerberos on all service accounts; enforce strong password policy (25+ chars) for service accounts; consider gMSAs to remove manual password management. |
| **References** | MITRE T1558.003, Microsoft Kerberos docs, adsecurity.org |

---

### ✅ Good Finding — Tomcat Manager Default Credentials

| Field | Content |
|---|---|
| **Risk** | High — CVSS 9.5 |
| **Description** | The Apache Tomcat Manager interface at `192.168.195.205:8080/manager` accepts default credentials (`tomcat:tomcat`), allowing unauthenticated remote code execution via WAR file deployment. |
| **Impact** | Full server compromise and potential lateral movement from the compromised host. |
| **Affected Systems** | `192.168.195.205` |
| **Remediation** | Change all default credentials immediately; restrict `/manager` access to trusted IPs via `context.xml`; disable the Manager app entirely if not needed in production. |
| **References** | Apache Tomcat Hardening Guide, OWASP Testing Guide OTG-AUTHN-002 |

---

### ❌ Bad Finding — Kerberoasting (poorly written version)

**Problems with this version:**
- Sloppy CWE link formatting
- No CVSS score filled in
- Description doesn't explain the root cause
- Impact is vague ("attacker can compromise the network")
- Remediation says "delete SPN accounts and use gMSA" with zero explanation or steps

**What's missing:** A reader who doesn't already know what Kerberoasting is won't understand the risk or how to fix it. The finding fails its primary job.

---

## Methodology — Writing a Finding End-to-End

1. **Identify the finding** — note the exact host, service, version, and config that's vulnerable.
2. **Capture evidence as you go** — don't try to reconstruct after the fact. Screenshot each step in sequence.
3. **Draft the description** — explain it to a smart non-specialist. If they'd need to Google every other word, rewrite it.
4. **Assess the impact** — think about the blast radius: what can an attacker do *next* after exploiting this?
5. **Write reproduction steps** — one figure per action, narrative between each.
6. **Write remediation** — be specific. Include exact commands, config paths, or values. Add a risk caveat if the fix is disruptive.
7. **Pick references** — aim for 2 solid links. Write your own if you have a blog.
8. **Assign severity / CVSS** — don't leave this blank if your template has it.
9. **Redact credentials** in all screenshots and command output before finalising.
10. **Review for defensibility** — ask: "Could this evidence be argued away?" If yes, add more proof.

---

## Lab — Questions & Answers

| Q | Answer | Notes |
|---|--------|-------|
| "An attacker can own your whole entire network cause your DC is way out of date. You should really fix that!" — Good or Bad remediation recommendation? | **Bad** | Vague, unprofessional, no specifics on what's out of date, no remediation steps, no risk context, inappropriate language for a client deliverable |

---

## Optional Exercise

Connect to the WriteHat instance and practice:
- Adding findings to the database
- Building and generating a report
- Using the pre-populated findings categories
- Writing up findings from the Obsidian notebook evidence

```
xfreerdp /u:htb-student /p:'HTB_@cademy_stdnt!' /v:<TARGET_IP>
# Then browse to: https://<TARGET_IP>/login
# Credentials: htb-student / HTB_@cademy_stdnt!
```

> ⚠️ WriteHat data does not persist after the target expires — save anything you write locally before the session ends.

---

## Key Takeaways

- **Vague remediation = useless finding.** If the client can't act on it without 4 hours of Googling, rewrite it.
- **Evidence must be legally defensible.** A login popup proves Basic Auth is in use — it does not prove cleartext transmission. Add the Wireshark capture.
- **Always show the target IP or URL** in GUI screenshots — never just the vuln on its own.
- **Stock findings must be customised.** A default credential finding on a printer has completely different risk than the same finding on an HVAC controller.
- **Cost-aware remediation builds trust.** Always offer at least one path that doesn't require buying new software.
- **Redact credentials** in all evidence — the report will be passed around to many audiences.
- **WriteHat** (by Black Lantern Security) is a solid tool for building a findings database and templating reports — worth exploring for your personal workflow.

---

## Gotchas

- Screenshotting Burp Suite for a web finding looks unprofessional and is not copy-pasteable — use raw request text instead.
- Forgetting to turn off browser bookmarks / personal extensions in screenshots looks sloppy in a client deliverable.
- Writing remediation that recommends a risky change (registry edits, GPO pushes) **without a caution note** can expose you to liability if the client blindly follows it and breaks something.
- "Not mandatory" fields like CVSS still need to be filled in if your report template includes them — leaving them blank signals laziness.
- WriteHat session data is lost when the target expires — don't rely on it as your only copy.
