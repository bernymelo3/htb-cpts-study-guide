# NOTE — Certificate Transparency Logs

## ID
209

## Module
Information Gathering – Web Edition

## Kind
theory

## Title
Section 10 — Certificate Transparency Logs

## Description
CT logs are public, append-only records of every SSL/TLS certificate issued. Every cert leaks its SAN list, which means every cert is a free subdomain enumerator. Includes the `crt.sh` JSON+jq+grep one-liner.

## Tags
ct-logs, crt.sh, censys, passive-recon, subdomain-discovery, ssl, tls, san

## Commands
- `curl -s "https://crt.sh/?q=facebook.com&output=json" | jq -r '.[] | .name_value' | sort -u`
- `curl -s "https://crt.sh/?q=facebook.com&output=json" | jq -r '.[] | select(.name_value | contains("dev")) | .name_value' | sort -u`

## TL;DR — What's Important
- Every SSL/TLS cert issued by a public CA goes into multiple CT logs (mandatory since ~2018).
- Each cert has a SAN (Subject Alternative Name) list — that's a freebie subdomain dump.
- crt.sh has a JSON API — perfect for scripting (`jq` + `grep` + `sort -u`).
- Stealthy: zero packets to the target. Only downside vs brute force: you only see subdomains that were ever issued a public cert.

## Concept Overview
CT logs are public, append-only ledgers maintained by independent orgs. They were created to detect rogue/mis-issued certs — a transparency mechanism for the Web PKI. As a side effect, they're an enormous historical database of every domain+subdomain combination that's ever been TLS-protected.

## Why CT Logs Beat Brute-Force (sometimes)
| | Brute force | CT logs |
|---|---|---|
| Target traffic | Yes (DNS queries) | Zero |
| Coverage | Limited to wordlist | Limited to historically-issued certs |
| Finds dead/expired subs | No | Yes (huge plus) |
| Speed | Slow | Instant |
| Wildcard issues | Yes | Wildcard certs hide the actual list |

## Two Main Search Tools
| Tool | Strength |
|---|---|
| **crt.sh** | Free, public, JSON API, no login. Best for quick lookups. |
| **Censys** | Powerful filters, internet-wide cert+host data. Free tier with registration. |

## crt.sh JSON API Recipe
The whole point of `crt.sh` for recon is the `output=json` endpoint:
```bash
curl -s "https://crt.sh/?q=facebook.com&output=json" \
  | jq -r '.[] | .name_value' \
  | sort -u
```

Filter by keyword (e.g. only "dev" subdomains):
```bash
curl -s "https://crt.sh/?q=facebook.com&output=json" \
  | jq -r '.[] | select(.name_value | contains("dev")) | .name_value' \
  | sort -u
```

| Pipeline stage | What it does |
|---|---|
| `curl -s` | Silent fetch of the JSON dump |
| `jq -r '.[]'` | Iterate each cert entry |
| `select(.name_value \| contains("dev"))` | Keyword filter |
| `.name_value` | The SAN field — one or more domains per cert |
| `sort -u` | Dedup |

## Why CT Logs Matter Beyond Recon
- **Early rogue-cert detection** — defenders monitor for unauthorized certs in their domains.
- **CA accountability** — public log entries hold CAs accountable for mis-issuance.
- **Strengthens Web PKI** — provides the audit trail that browser CT enforcement relies on.

## Key Takeaways
- crt.sh is the cheapest first-pass for subdomain discovery. Always run it before brute force.
- Use the JSON API + jq for scripting; the HTML UI is for one-off browsing.
- Wildcard certs (`*.example.com`) collapse the SAN list — supplement with brute force when you see lots of wildcards.
- CT logs include expired/decommissioned subdomains — those legacy hostnames sometimes still resolve and host vulnerable apps.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← [[09-virtual-hosts]] | [[11-fingerprinting]] →
<!-- AUTO-LINKS-END -->
