# NOTE — DNS Zone Transfers

## ID
207

## Module
Information Gathering – Web Edition

## Kind
lab

## Title
Section 8 — DNS Zone Transfers

## Description
Wholesale copy of all DNS records from a primary NS to a secondary (AXFR). When misconfigured to allow anyone, it dumps the entire zone — every subdomain, IP, MX, NS — in one query. Demonstrated against the public test target `zonetransfer.me` and the lab's `inlanefreight.htb`.

## Tags
zone-transfer, axfr, dns, dig, lab, misconfiguration, soa, secondary-server

## Commands
- `dig axfr @nsztm1.digi.ninja zonetransfer.me`
- `dig axfr inlanefreight.htb @<target_ns_ip>`
- `dig axfr inlanefreight.htb @<target_ns_ip> | grep "ftp.admin.inlanefreight.htb"`
- `dig axfr inlanefreight.htb @<target_ns_ip> | grep "10.10.200"`

## What This Section Covers
Zone transfers (AXFR) are the legitimate replication mechanism between primary and secondary DNS servers. If a primary is misconfigured to allow AXFR from any client, an attacker can dump the *entire* zone file in a single query — every subdomain, every internal IP, every MX/NS record.

## How AXFR Works (the legit flow)
1. **Secondary requests** AXFR from primary.
2. **Primary sends SOA record** — secondary checks serial number to know if data is stale.
3. **Primary streams all RRs** — A, AAAA, MX, CNAME, NS, etc.
4. **Primary sends end-of-transfer signal**.
5. **Secondary ACKs** completion.

When access controls are absent or misconfigured, step 1 is open to anyone.

## What an Open AXFR Reveals
| Data | Why it matters |
|---|---|
| All subdomains | Including hidden / internal / dev / admin ones not linked from `www.` |
| Internal IPs | RFC1918 ranges (`10.x`, `192.168.x`) embedded in zones for split-horizon DNS |
| MX/NS records | Hosting provider, mail provider, dev infra |
| Service hostnames | `ftp.`, `vpn.`, `wsus.`, `dc1.`, etc. |

## Methodology
1. **Find the authoritative NS** — `dig NS <domain>`.
2. **Try AXFR against each NS** — `dig axfr @<ns> <domain>`.
3. **If allowed** — output is dozens to hundreds of RRs. Pipe through `grep` to find specific hosts.
4. **If denied** — you'll see `Transfer failed` or no records returned.
5. **Cross-reference** internal IPs (10.x ranges) with later steps — these become next-hop targets after a foothold.

## Demo — `zonetransfer.me`
A purpose-built lab by Digi.Ninja. Open AXFR by design:
```bash
dig axfr @nsztm1.digi.ninja zonetransfer.me
```
Returns ~50 RRs including SOA, A, MX, TXT, NS, SRV, AFSDB, PTR.

## Remediation (defender perspective)
Modern DNS servers restrict AXFR to a hardcoded list of secondary IPs (or use TSIG keys). The vulnerability persists in legacy / misconfigured environments — always worth a one-shot test.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — How many DNS records are retrieved from a zone transfer of `inlanefreight.htb`? | **{see lab — answer redacted in walkthrough}** | `dig axfr inlanefreight.htb @<STMIP>` — count records (output shows `XFR size: N records`) |
| Q2 — IP for `ftp.admin.inlanefreight.htb` | **{see lab — answer redacted in walkthrough}** | `dig axfr inlanefreight.htb @<STMIP> \| grep "ftp.admin.inlanefreight.htb"` |
| Q3 — Largest IP allocated within the `10.10.200.x` range | **{see lab — answer redacted in walkthrough}** | `dig axfr inlanefreight.htb @<STMIP> \| grep "10.10.200"` — pick the highest last octet |

## Key Takeaways
- AXFR is the cheapest, highest-payoff DNS recon attempt: if it works, you skip subdomain brute-force entirely.
- Always test against *every* NS — sometimes only one of them is misconfigured.
- The output gives you internal IP ranges for free — a head-start on internal pivoting later.

## Gotchas
- "No records" can mean either denied transfer or the NS is down — try a normal `dig <domain>` against the same NS to confirm reachability.
- Some servers return a partial transfer (just SOA) when restricted — that's a deny, not a success.
- `axfr` uses TCP, not UDP — make sure port 53/tcp is reachable.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← [[07-subdomain-bruteforcing]] | [[09-virtual-hosts]] →
<!-- AUTO-LINKS-END -->
