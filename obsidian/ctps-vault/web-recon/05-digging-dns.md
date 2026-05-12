# NOTE — Digging DNS

## ID
204

## Module
Information Gathering – Web Edition

## Kind
lab

## Title
Section 5 — Digging DNS

## Description
Practical use of `dig` to query A, MX, NS, TXT, CNAME, SOA, PTR, and ANY records, plus the `+short`, `+trace`, `-x`, and `@<server>` modifiers. Also covers other DNS tooling (nslookup, host, dnsenum, fierce, dnsrecon, theHarvester).

## Tags
dns, dig, lab, ptr, mx, reverse-dns, nslookup, host, dnsrecon, theharvester

## Commands
- `dig domain.com`
- `dig domain.com A`
- `dig domain.com MX`
- `dig domain.com NS`
- `dig domain.com TXT`
- `dig domain.com SOA`
- `dig +short domain.com`
- `dig +noall +answer domain.com`
- `dig +trace domain.com`
- `dig @1.1.1.1 domain.com`
- `dig -x 134.209.24.248`
- `dig domain.com ANY`

## What This Section Covers
`dig` (Domain Information Groper) is the workhorse for DNS recon. This section covers the common query types, output sections (header / question / answer / footer), and the modifiers you'll actually use in engagements.

## Methodology
1. **Default lookup** (`dig <domain>`) returns the A record by default. Verbose — shows header, question, answer, footer.
2. **Type-specific** (`dig <domain> MX`) for mail; `NS` for delegation; `TXT` for SaaS leaks.
3. **`+short`** strips everything but the answer — script-friendly.
4. **`+noall +answer`** keeps just the answer section but with full RR formatting.
5. **`@<server>`** queries a specific resolver — useful for AXFR ([[08-dns-zone-transfers]]) or comparing public vs internal DNS views.
6. **`-x <ip>`** does reverse PTR lookup — `dig` builds the `.in-addr.arpa.` query for you.
7. **`+trace`** shows the full recursive walk root → TLD → authoritative.

## `dig` Command Cheat-Sheet
| Command | Description |
|---|---|
| `dig domain.com` | Default A lookup |
| `dig domain.com A` | IPv4 |
| `dig domain.com AAAA` | IPv6 |
| `dig domain.com MX` | Mail servers |
| `dig domain.com NS` | Authoritative NS |
| `dig domain.com TXT` | TXT records (SPF/DKIM/verification) |
| `dig domain.com CNAME` | Canonical name |
| `dig domain.com SOA` | Start of Authority |
| `dig @1.1.1.1 domain.com` | Use specific resolver |
| `dig +trace domain.com` | Full recursive trace |
| `dig -x 192.168.1.1` | Reverse PTR lookup |
| `dig +short domain.com` | Concise answer only |
| `dig +noall +answer domain.com` | Answer section only |
| `dig domain.com ANY` | All records (often filtered — RFC 8482) |

## Reading the Output
| Section | What it tells you |
|---|---|
| Header | `opcode`, `status` (NOERROR / NXDOMAIN / SERVFAIL), query ID |
| Flags | `qr` (response), `rd` (recursion desired), `ad` (authentic data), `aa` (authoritative answer) |
| Question | What you asked |
| Answer | The records returned (with TTL) |
| Footer | Query time, server queried, timestamp, message size |

## Other DNS Tools
| Tool | Use |
|---|---|
| `dig` | Verbose, scriptable, supports every record type |
| `nslookup` | Simpler (mostly A/AAAA/MX), interactive mode |
| `host` | One-liner output for A/AAAA/MX |
| `dnsenum` | Automated enum + brute force + zone-transfer attempt |
| `fierce` | Recursive subdomain discovery + wildcard detection |
| `dnsrecon` | Multi-technique enum, multiple output formats |
| `theHarvester` | OSINT — emails + DNS + employees from public sources |

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — IP address that maps to `inlanefreight.com` | **{see lab — answer redacted in walkthrough}** | `dig +short inlanefreight.com` |
| Q2 — Domain returned for PTR of `134.209.24.248` | **{see lab — answer redacted in walkthrough}** | `dig -x 134.209.24.248` |
| Q3 — Full domain returned when querying MX records of `facebook.com` | **{see lab — answer redacted in walkthrough}** | `dig MX facebook.com` |

## Key Takeaways
- `dig +short` is your friend for scripts and one-liners.
- `dig -x` skips manual `.in-addr.arpa.` construction for reverse lookups.
- `ANY` queries are increasingly filtered (RFC 8482) — fall back to per-type queries.
- Always specify the resolver (`@1.1.1.1`) when comparing public vs internal DNS views.

## Gotchas
- Excessive `dig` traffic *can* be detected by upstream resolvers — respect rate limits.
- `dig +trace` queries direct from root — bypasses the local resolver cache, useful when you suspect cache poisoning or stale records.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← [[04-dns]] | [[06-subdomains]] →
<!-- AUTO-LINKS-END -->
