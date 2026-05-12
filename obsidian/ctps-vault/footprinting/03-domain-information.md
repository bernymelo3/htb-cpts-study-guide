# NOTE — Domain Information

## ID
15

## Module
Footprinting

## Kind
notes

## Title
Section 3 — Domain Information

## Description
Passive infrastructure recon: SSL cert SANs, crt.sh certificate transparency, Shodan, and DNS records (A/MX/NS/TXT/SOA) — turn a single domain into a full external footprint.

## Tags
domain, dns, crt.sh, certificate-transparency, shodan, dig, txt-records, passive-recon

## Commands
- `curl -s https://crt.sh/\?q\=<domain>\&output\=json | jq .`
- `curl -s https://crt.sh/\?q\=<domain>\&output\=json | jq . | grep name | cut -d":" -f2 | grep -v "CN=" | cut -d'"' -f2 | awk '{gsub(/\\n/,"\n");}1;' | sort -u`
- `for i in $(cat subdomainlist); do host $i | grep "has address" | grep <domain> | cut -d" " -f1,4; done`
- `for i in $(cat ip-addresses.txt); do shodan host $i; done`
- `dig any <domain>`

## What This Section Covers
Stay in "customer/visitor" mode — no active scans against the company. You inspect the company website, its SSL cert, public certificate transparency logs, Shodan, and DNS records to assemble a list of subdomains, IPs, hosting providers, and third-party services.

## Methodology
1. **Read the company website like a developer.** What services do they offer? Each service implies a stack (web app → frameworks/databases; IoT → MQTT/cloud; etc.).
2. **Look at the SSL certificate's SAN list** on the main site — one cert often covers multiple subdomains.
3. **Query crt.sh** for certificate transparency log entries — every cert ever issued for the domain. Filter unique subdomains via `jq + grep + sort -u`.
4. **Resolve subdomains to IPs**, then split into "hosted by company" vs "hosted by third-party (AWS, Google, etc.)". Only test hosts you have permission for.
5. **Run IPs through Shodan** to see open ports and banners passively.
6. **`dig any <domain>`** to dump A, MX, NS, TXT, SOA in one shot — note hidden gold in TXT records.

## DNS Record Reference
| Record | What it tells you |
|--------|-------------------|
| A | IPv4 of (sub)domain |
| MX | Mail server — `google.com` MX = Gmail, internal MX = self-hosted (worth testing) |
| NS | Name servers — often reveal the hosting provider |
| TXT | Verification keys, SPF/DKIM/DMARC — leaks third-party services in use |
| SOA | Admin email + zone metadata |

## TXT Records — The Goldmine
A single SPF/verification dump can reveal:
- **Atlassian** → `atlassian-domain-verification=...` → Confluence/Jira/Bitbucket likely in use
- **Google** → `google-site-verification=...` + Gmail MX → GDrive, possibly leaked links
- **Outlook / `MS=...`** → Office 365, OneDrive, Azure storage (incl. Azure Files over SMB)
- **Mailgun** → API/webhooks worth probing for IDOR/SSRF
- **LogMeIn** → centralised remote access — single account compromise = wide blast radius
- **`MS=ms<id>`** → often matches the registrar/hosting account username (e.g. INWX)

SPF `ip4:` entries leak the company's actual sending IPs — record these as in-scope candidates.

## Lab — Questions & Answers
*(No lab in this section — pure theory/recon walk-through.)*

## Key Takeaways
- crt.sh is your fastest source of subdomains. No active probes hit the target.
- Always separate "company-hosted" from "third-party-hosted" before scanning — third parties are out of scope without explicit consent.
- TXT records leak the company's whole SaaS stack for free.
- Shodan = passive port scan via someone else's infrastructure.

## Gotchas
- AWS/GCP/Azure hostnames in your IP resolution = NOT in scope unless contracted.
- `dig any` is increasingly filtered (RFC 8482) — modern resolvers may return only HINFO. Fall back to `dig <domain> A`, `MX`, `TXT`, `NS` separately.
- A wildcard cert (`*.<domain>`) on crt.sh hides the actual subdomain list — supplement with brute-force.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[02-enumeration-methodology]] | [[04-cloud-resources]] →
<!-- AUTO-LINKS-END -->
