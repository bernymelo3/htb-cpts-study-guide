## ID
602

## Module
Documentation & Reporting

## Kind
notes

## Title
Section 4 — Components of a Report

## Description
Covers every structural component of a professional pentest report — attack chain writing, executive summary best practices, findings section, appendices, and report type differences — using a full sample AD compromise chain as a walkthrough.

## Tags
documentation, reporting, executive-summary, attack-chain, findings, active-directory

## Commands
- `sudo responder -I <INTERFACE> -wrfv`
- `hashcat -m 5600 <HASH_FILE> /usr/share/wordlists/rockyou.txt`
- `GetUserSPNs.py <DOMAIN>/<USER> -dc-ip <DC_IP>`
- `GetUserSPNs.py <DOMAIN>/<USER> -dc-ip <DC_IP> -request-user <TARGET_USER>`
- `sudo bloodhound-python -u '<USER>' -p '<PASS>' -d <DOMAIN> -ns <DC_IP> -c All`
- `hashcat -m 13100 <TGS_FILE> /usr/share/wordlists/rockyou.txt`
- `crackmapexec smb <TARGET_IP> -u <USER> -p <PASS> --lsa`
- `secretsdump.py <DOMAIN>/<USER>@<DC_IP> -hashes <LM>:<NT> -just-dc-ntlm`
- `.\Rubeus.exe triage`
- `.\Rubeus.exe dump /luid:<LUID> /service:krbtgt`
- `.\Rubeus.exe ptt /ticket:<BASE64_TICKET>`
- `.\mimikatz.exe` → `lsadump::dcsync /user:<DOMAIN>\administrator`

## What This Section Covers
Defines the full anatomy of a professional internal penetration test report, walking through each component from attack chain to appendices. Uses a complete sample AD compromise chain (Responder → Kerberoasting → LSA secrets → Pass-the-Ticket → DCSync) to demonstrate how findings connect and how to document them. Also covers executive summary writing principles and the distinction between static and dynamic appendices.

## Report Structure Reference

| Component | Audience | Purpose |
|---|---|---|
| Attack Chain | Technical + Executive | Shows full compromise path; connects findings together |
| Executive Summary | Non-technical (C-suite, board) | Communicates risk in plain language; drives budget decisions |
| Summary of Recommendations | Management | Short/medium/long-term remediation roadmap |
| Findings | Technical teams | Evidence, reproduction steps, remediation advice |
| Appendices (static) | Auditors, compliance | Scope, methodology, severity ratings, bios |
| Appendices (dynamic) | Client security teams | Payloads, compromised creds, config changes, password analysis |

## Sample Attack Chain — INLANEFREIGHT.LOCAL
> Grey box, non-evasive internal pentest → full AD domain compromise

| Step | Tool | Technique | Result |
|---|---|---|---|
| 1 | Responder | NBT-NS/LLMNR spoofing | NTLMv2 hash for `bsmith` |
| 2 | Hashcat (`-m 5600`) | Offline hash cracking | Cleartext password → domain foothold |
| 3 | BloodHound.py | AD enumeration | Identified SPNs; `mssqlsvc` has local admin on SQL01 |
| 4 | GetUserSPNs.py | Kerberoasting (targeted) | TGS ticket for `mssqlsvc` |
| 5 | Hashcat (`-m 13100`) | Offline TGS cracking | Cleartext password for `mssqlsvc` |
| 6 | CrackMapExec `--lsa` | LSA secrets dump on SQL01 | Cleartext creds for `srvadmin` (autologon) |
| 7 | RDP to MS01 | Lateral movement | Found `pramirez` logged in; has DCSync rights via BloodHound |
| 8 | Rubeus `triage` + `dump` | Kerberos ticket extraction | Grabbed TGT for `pramirez` (LUID `0x1a8b19`) |
| 9 | Rubeus `ptt` | Pass-the-Ticket | Authenticated as `pramirez` |
| 10 | Mimikatz `dcsync` | DCSync attack | NTLM hash for `Administrator` → domain compromise |
| 11 | CrackMapExec | Hash auth to DC | Confirmed Domain Admin access |
| 12 | secretsdump.py | NTDS dump | All domain hashes retrieved |

## Executive Summary — Do's and Don'ts

### DO
- **Be specific with numbers** — "25 occurrences" not "several" or "multiple"
- **Keep it to 1.5–2 pages max** — collapse findings into higher-level categories
- **Describe what you accessed in plain terms** — not "Domain Admin" but "access to HR documents, banking systems, and critical assets"
- **Describe the process breakdown** — not just "patch X" but "what broke down that let a 5-year-old vuln go unpatched in 25% of the environment?"
- **Acknowledge what they're doing well** — e.g., mature patch management; endears you to the sysadmin team
- **Use qualifiers when making inferences** — "seemed to go mostly unnoticed" not "was not detected"

### DO NOT
- **Name specific vendors** — say "EDR solution" not "CrowdStrike"; say "log aggregation" not "Splunk"
- **Use acronyms** — no SNMP, MitM, TGS, LLMNR, etc. IP and VPN are borderline acceptable
- **Reference other sections** — executive readers won't scroll back and forth
- **Spend time on low-impact findings** — focus attention on high-severity issues
- **Write in absolutes based on assumptions** — "begin documenting hardening" implies they haven't; say "review and address gaps in..."

### Vocabulary Translations
| Technical Term | Plain Language Alternative |
|---|---|
| VPN / SSH | Protocol used for secure remote administration |
| SSL/TLS | Technology that facilitates secure web browsing |
| Hash | Output from an algorithm used to validate file integrity |
| Password Spraying | Attempting a single guessable password across many user accounts |
| Password Cracking | Converting the cryptographic form of a password back to human-readable form |
| Buffer overflow / RCE vuln | An attack that resulted in remote command execution on the host |
| OSINT | Hunting data about a company using search engines and public sources without touching their network |
| SQL injection / XSS | A vulnerability where unvalidated user input manipulates the application's logic |

## Findings Prioritization
- Disclose everything but **focus effort on high-impact issues** (RCE, sensitive data disclosure)
- Consolidate low-impact noise into categories rather than validating each individually (e.g., "35 SSL/TLS variants" → one consolidated finding)
- Use senior team members / mentors to avoid wasting time on rabbit holes and false positives

## Summary of Recommendations Structure
- Tie each recommendation to a specific finding
- Short-term: actionable fixes (patch, config change, rotate creds)
- Long-term: process improvements (patch management program, SDLC review, periodic VA)
- Some findings warrant both (e.g., missing patch = short-term: push patch; long-term: fix the patch management process that let it happen)

## Static Appendices (every report)
| Appendix | Content |
|---|---|
| Scope | In-scope IPs, URLs, facilities — required by auditors |
| Methodology | Repeatable process documentation |
| Severity Ratings | Criteria for each severity level; must be defensible |
| Biographies | Consultant qualifications — required for PCI assessments |

## Dynamic Appendices (as applicable)
| Appendix | When to Include |
|---|---|
| Exploitation Attempts & Payloads | Always — lists tools used, payloads dropped, file paths, hashes |
| Compromised Credentials | When accounts were compromised (list them or say "all domain accounts") |
| Configuration Changes | Any change made in the client environment with revert instructions |
| Additional Affected Scope | When affected host lists are too long for the finding itself |
| Information Gathering | External pentests — whois, subdomains, breach data, SSL config, open ports |
| Domain Password Analysis | When NTDS was dumped — use DPAT for stats (% cracked, top passwords, privileged accounts cracked) |

## Report Type Differences
| Report Type | Key Differences |
|---|---|
| Internal Pentest (AD compromise) | Full attack chain, compromised creds appendix, domain password analysis |
| External Pentest (no internal compromise) | Focus on info gathering, OSINT, externally exposed services; no attack chain |
| Web App Assessment (WASA) | Executive summary + findings; emphasis on OWASP Top 10 |
| Red Team / Physical / Social Engineering | Narrative format |

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| What component should be written in simple, non-technical language? | Executive Summary | Defined in the Executive Summary section |
| Is it good practice to name/recommend specific vendors in the Executive Summary? | False | Do Not list explicitly states: avoid naming specific vendors |

## Key Takeaways
- **The attack chain exists to connect findings** — a medium finding combined with two others can become high-risk; the chain shows this to the client and justifies severity ratings.
- **Evidence from the attack chain is reusable** — format it once, copy/paste into individual findings; don't document the same thing twice.
- **The executive summary is the most important page** — if the non-technical reader loses interest here, the rest of the report becomes worthless and the client may not fund remediation.
- **Never write the executive summary in absolutes based on assumptions** — qualifiers like "seemed," "may indicate," and "appeared to go unnoticed" protect your professional integrity.
- **Recommendations need to address the process failure, not just the symptom** — 500 accounts with `Welcome1!` means the help desk needs better initial password processes, not just 500 password resets.
- **Dynamic appendices are part of your professional obligation** — clients need your payload list and config changes to clean up after you and to differentiate your activity from a real attacker during incident response.

## Gotchas
- Don't say "all domain accounts compromised" as a throwaway — if the client is PCI-scoped, listing every account may actually be required; confirm requirements at kickoff.
- DPAT password analysis report: only include it if you actually exhausted cracking attempts properly — a low crack rate from a lazy wordlist undermines the finding narrative.
- If a finding list has 20 items and no remediation prioritization, the client will ask anyway — build the Summary of Recommendations proactively to control the narrative.
- Post-remediation scope: never run new vuln scans as part of retest — you'll find new findings, scope spirals, and you'll end up in an endless loop.
