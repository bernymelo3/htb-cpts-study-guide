# NOTE — DNS

## ID
203

## Module
Information Gathering – Web Edition

## Kind
theory

## Title
Section 4 — DNS

## Description
DNS fundamentals for recon: zones, resolvers, the recursive lookup chain (root → TLD → authoritative), the `/etc/hosts` override, and the full record-type cheat-sheet (A, AAAA, CNAME, MX, NS, TXT, SOA, SRV, PTR).

## Tags
dns, theory, zone-file, record-types, hosts-file, resolver, recursive-lookup

## TL;DR — What's Important
- DNS = the internet's phonebook: hostname → IP, plus mail (MX), aliases (CNAME), text (TXT), reverse (PTR).
- Recursive lookup chain: client → resolver → root NS → TLD NS → authoritative NS → answer cached back down.
- `/etc/hosts` (Linux/Mac) or `C:\Windows\System32\drivers\etc\hosts` (Windows) bypasses DNS — required for vhost testing where the subdomain has no public DNS record.
- Zone = administrative chunk of namespace. Zone file = the records for that zone, served by an authoritative NS.

## Concept Overview
DNS translates `www.example.com` → `192.0.2.1` so humans don't have to memorise IPs. Without it the web is unusable. For recon, DNS records are an explicit map of a target's external infrastructure: every subdomain you find is a potential foothold.

## How a DNS Lookup Works
1. **Browser asks the OS** — checks local cache.
2. **OS asks the resolver** (usually ISP, sometimes 1.1.1.1 / 8.8.8.8).
3. **Resolver checks its cache**, otherwise starts recursive walk:
4. **Root server** → points to TLD server (`.com`).
5. **TLD server** → points to authoritative server for the domain.
6. **Authoritative server** → returns the record (e.g. A 192.0.2.1).
7. **Resolver caches** the answer (TTL-bound) and returns it to client.

## The hosts file
Bypasses DNS — direct hostname→IP override. Format:
```
<IP Address>    <Hostname> [<Alias> ...]
127.0.0.1       localhost
192.168.1.10    devserver.local
```

| OS | Path |
|---|---|
| Linux / macOS | `/etc/hosts` |
| Windows | `C:\Windows\System32\drivers\etc\hosts` |

Used in this module for: vhost testing (when the subdomain has no public A record), local dev redirection, blocking domains.

## Key DNS Concepts
| Concept | Description |
|---|---|
| Domain Name | Human label (`www.example.com`) |
| IP Address | Numeric ID (`192.0.2.1`) |
| Resolver | Server that walks the DNS tree on the client's behalf |
| Root Name Server | 13 worldwide (a-m).root-servers.net |
| TLD Name Server | Manages a TLD (Verisign for `.com`, PIR for `.org`) |
| Authoritative NS | Final answer for a domain |
| Zone | Admin chunk of namespace (e.g. `example.com` and its subdomains) |
| Zone File | Text file on the authoritative NS holding the records |

## DNS Record Types (the ones you'll actually use)
| Type | Full name | What it does | Zone-file example |
|---|---|---|---|
| **A** | Address | Hostname → IPv4 | `www IN A 192.0.2.1` |
| **AAAA** | IPv6 Address | Hostname → IPv6 | `www IN AAAA 2001:db8::1` |
| **CNAME** | Canonical Name | Alias → another hostname | `blog IN CNAME web.example.net.` |
| **MX** | Mail Exchange | Mail server for the domain | `@ IN MX 10 mail.example.com.` |
| **NS** | Name Server | Authoritative NS for the zone | `@ IN NS ns1.example.com.` |
| **TXT** | Text | Arbitrary text — SPF, DKIM, verification keys | `@ IN TXT "v=spf1 mx -all"` |
| **SOA** | Start of Authority | Zone admin metadata (serial, refresh, etc.) | `@ IN SOA ns1 admin 2024060401 ...` |
| **SRV** | Service | Service+port pointer | `_sip._udp IN SRV 10 5 5060 sipserver.example.com.` |
| **PTR** | Pointer | Reverse DNS (IP → hostname) | `1.2.0.192.in-addr.arpa. IN PTR www.example.com.` |

The `IN` class field stands for *Internet* — basically all modern DNS uses it.

## Why It Matters in Recon
- **Uncovers assets** — subdomains, mail servers, NS records.
- **Maps the network** — A records → IPs → hosting provider; NS records → DNS provider.
- **Detects changes over time** — a new subdomain appearing (`vpn.example.com`) signals a new entry point.
- **Reveals SaaS stack via TXT** — `_1password=`, `google-site-verification=`, `MS=` all leak third-party services.

## Key Takeaways
- Every record type is a potential intel source — don't just query A.
- TXT records are the goldmine for passive recon (see [[../footprinting/03-domain-information|footprinting/domain-information]]).
- `/etc/hosts` is mandatory for vhost work later in this module ([[09-virtual-hosts]]).

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← [[03-utilizing-whois]] | [[05-digging-dns]] →
<!-- AUTO-LINKS-END -->
