# NOTE — Utilizing WHOIS

## ID
202

## Module
Information Gathering – Web Edition

## Kind
lab

## Title
Section 3 — Utilizing WHOIS

## Description
Practical use of `whois`: install, run, and grep relevant fields (IANA ID, admin email) for fast pre-engagement intel. Includes three real-world scenarios (phishing investigation, malware C2, threat-intel report).

## Tags
whois, lab, iana-id, admin-email, grep, phishing, c2, threat-intel

## Commands
- `sudo apt update && sudo apt install whois -y`
- `whois facebook.com`
- `whois <domain> | grep IANA`
- `whois <domain> | grep "admin"`

## What This Section Covers
The CLI `whois` tool plus three short case studies showing how WHOIS data drives real triage decisions: spotting a freshly-registered phishing domain, tracking a malware C2's bulletproof host, and clustering a threat actor's infrastructure by registration patterns.

## Methodology
1. **Install** if missing: `sudo apt install whois -y`.
2. **Raw lookup**: `whois <domain>` — full dump.
3. **Filter for the field you want**: pipe through `grep` for IANA ID, contacts, name servers, dates, etc.
4. **Cross-check** registrar + dates + NS for anomalies (recent registration + privacy + shady NS = phishing red flag).
5. **For historical data**, use WhoisFreaks or similar.

## Field Cheat-Sheet
| Want | Grep filter |
|---|---|
| Registrar IANA ID | `grep IANA` |
| Admin contact email | `grep "admin"` |
| Name servers | `grep -i "name server"` |
| Creation/expiry dates | `grep -iE "creation\|expir"` |
| Registrant org | `grep -i "registrant"` |
| Abuse contact | `grep -i "abuse"` |

## Three Reading-Between-the-Lines Scenarios
| Scenario | Telltale signs in WHOIS |
|---|---|
| **Phishing investigation** | Domain registered days ago + privacy-shielded registrant + bulletproof-host NS |
| **Malware C2 analysis** | Free-email registrant + high-cybercrime jurisdiction + lax registrar |
| **Threat-intel cluster** | Multiple domains share NS + registered in clusters before campaigns + repeat aliases |

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Registrar IANA ID for `paypal.com` | **{see lab — answer redacted in walkthrough}** | `whois paypal.com \| grep IANA` |
| Q2 — Admin email contact for `tesla.com` | **{see lab — answer redacted in walkthrough}** | `whois tesla.com \| grep "admin"` |

## Key Takeaways
- `whois` returns a wall of text — always pipe through `grep` for the specific field you need.
- The IANA ID identifies the registrar uniquely (better than the registrar name string, which varies).
- `Admin Email`, `Registrant Email`, and `Tech Email` are three separate fields — grep for the literal label, not just "email".

## Gotchas
- TLD-specific WHOIS servers can return wildly different formats. The `whois` CLI usually follows referrals automatically — but some ccTLDs need an explicit server (e.g. `whois -h whois.nic.uk`).
- Privacy services are now the default for `.com` — expect `Registrant Name: REDACTED FOR PRIVACY` on most modern registrations.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← [[02-whois]] | [[04-dns]] →
<!-- AUTO-LINKS-END -->
