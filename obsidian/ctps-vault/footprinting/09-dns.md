# NOTE — DNS

## ID
25

## Module
Footprinting

## Kind
notes

## Title
Section 9 — DNS

## Description
DNS server enumeration: NS/ANY/AXFR queries with dig, BIND version disclosure via CHAOS class, AXFR zone transfer abuse, and subdomain brute-forcing with DNSenum + SecLists.

## Tags
dns, bind, dig, axfr, zone-transfer, subdomain-bruteforce, dnsenum, named.conf

## Commands
- `dig ns <domain> @<DNS-IP>` — list authoritative NS
- `dig any <domain> @<DNS-IP>` — dump all available records
- `dig CH TXT version.bind <DNS-IP>` — disclose BIND version
- `dig axfr <domain> @<DNS-IP>` — full zone transfer (when allowed)
- `dig axfr internal.<domain> @<DNS-IP>` — try **internal** zones too
- `for sub in $(cat /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt); do dig $sub.<domain> @<DNS-IP> | grep -v ';\|SOA' | sed -r '/^\s*$/d' | grep $sub | tee -a subdomains.txt; done`
- `dnsenum --dnsserver <DNS-IP> --enum -p 0 -s 0 -o subdomains.txt -f /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt <domain>`
- `dig soa <domain>` — find admin email (replace `.` → `@` in first field)

## Concept Overview
DNS resolves domain names → IPs (and back, plus a lot more). Distributed across types: **root servers** (13 globally for TLDs), **authoritative** (binding source for a zone), **non-authoritative** (recursors), **caching**, **forwarding**, and local **resolvers**. Default is unencrypted; DoT/DoH/DNSCrypt offer encrypted variants. From an attacker's perspective DNS leaks: subdomains, mail infra, security/SaaS vendors, hosting providers, and sometimes whole internal zones.

## Record Types Reference
| Record | Use |
|--------|-----|
| `A` | IPv4 |
| `AAAA` | IPv6 |
| `MX` | Mail server (priority + name) |
| `NS` | Authoritative nameserver |
| `TXT` | Free text — SPF/DKIM/DMARC, vendor verifications |
| `CNAME` | Alias to another name |
| `PTR` | Reverse (IP → name) |
| `SOA` | Zone metadata + admin email (`.` → `@`) |

## BIND Configuration Basics
| File | Purpose |
|------|---------|
| `/etc/bind/named.conf` | Root config |
| `/etc/bind/named.conf.local` | Per-zone definitions |
| `/etc/bind/named.conf.options` | Global options |
| `/etc/bind/named.conf.log` | Logging |
| `/etc/bind/db.<domain>` | Forward zone file (BIND format) |
| `/etc/bind/db.<reverse>` | Reverse zone file (PTR records) |

Per-zone options override global options. Master = canonical source for a zone; slave = pulls from a master via AXFR.

### Dangerous Settings
| Option | Why it matters |
|--------|---------------|
| `allow-query { any; }` | Anyone may query — fine for public zones, bad for internal |
| `allow-recursion { any; }` | Open resolver — DNS amplification, cache poisoning |
| `allow-transfer { any; }` / wide subnet | Anyone gets full zone via AXFR |
| `zone-statistics` | Stats endpoint exposed |

## Methodology
1. **Enumerate NS** — `dig ns <domain> @<DNS-IP>`. Pivot to each NS — they may have different policies.
2. **Try `ANY`** — `dig any <domain> @<DNS-IP>`. Often returns SPF/TXT/MX/SOA in one shot.
3. **CHAOS-class version probe** — `dig CH TXT version.bind <DNS-IP>` — leaks BIND build (e.g. `9.10.6-P1-Debian` → CVEdetails lookup).
4. **Try AXFR** — `dig axfr <domain> @<DNS-IP>`. If allowed → entire zone (every subdomain, every IP).
5. **Try internal zones** — `dig axfr internal.<domain>`, `dev.<domain>`, `corp.<domain>`. Internal zones often have weaker `allow-transfer` policies.
6. **Brute-force subdomains** when AXFR is denied — for-loop with SecLists `subdomains-top1million-110000.txt`, or use `dnsenum`.

## AXFR — Zone Transfer
- Designed for master ↔ slave sync, secured by an `rndc-key` shared key.
- Misconfigured as `allow-transfer { any; };` it dumps the entire zone — every A/MX/NS/TXT for every name.
- Internal-zone AXFR success is the dream: dumps DC1, DC2, mail, vpn, ws01, ws02, wsus etc. — the complete internal map.

## Subdomain Brute-force One-liner
```bash
for sub in $(cat /opt/useful/seclists/Discovery/DNS/subdomains-top1million-110000.txt); do
  dig $sub.<domain> @<DNS-IP> | grep -v ';\|SOA' | sed -r '/^\s*$/d' | grep $sub | tee -a subdomains.txt
done
```

`dnsenum` automates this and tries AXFR + Bind-version + reverse lookups in one run.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — FQDN of the target DNS | **`inlanefreight.htb`** | `dig ANY inlanefreight.htb @<IP>` → returns SOA + NS for `inlanefreight.htb` |
| Q2 — TXT record exposed via zone transfer | **`HTB{DN5_z0N3_7r4N5F3r_iskdufhcnlu34}`** | `dig AXFR inlanefreight.htb @<IP>` succeeds → list internal subdomain → `dig AXFR internal.inlanefreight.htb @<IP>` returns the flag in a TXT record |
| Q3 — IPv4 of hostname `DC1` | **`10.129.34.16`** | From the same `internal.inlanefreight.htb` AXFR — record `dc1.internal.inlanefreight.htb. 604800 IN A 10.129.34.16` |
| Q4 — FQDN of host with last octet `.203` | **`win2k.dev.inlanefreight.htb`** | `dnsenum --dnsserver <IP> --enum -p 0 -s 0 -f /opt/useful/SecLists/Discovery/DNS/namelist.txt dev.inlanefreight.htb` → `win2k.dev.inlanefreight.htb 10.12.3.203` |

## Key Takeaways
- Always try AXFR — it costs one query and sometimes returns the whole infrastructure.
- The **second** AXFR attempt against `internal.<domain>` (or other obvious dev/corp zones) often succeeds when the public one fails.
- `dig CH TXT version.bind` is the single most effective version-disclosure probe — try it on every DNS server.
- SOA's first field encodes the admin email with `.` instead of `@`.

## Gotchas
- `dig any` is increasingly filtered (RFC 8482) — modern resolvers may return only HINFO. Fall back to specific record types.
- AXFR happens over **TCP/53**, not UDP. Filtered TCP/53 = no AXFR even if allowed.
- A `dig` against the wrong NS returns garbage — always specify `@<NS-IP>` from the NS list.
- Dead/orphaned subdomain entries point to IPs that may now belong to someone else (subdomain takeover risk to the company, in-scope for some pentests).

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[08-nfs]] | [[10-smtp]] →
<!-- AUTO-LINKS-END -->
