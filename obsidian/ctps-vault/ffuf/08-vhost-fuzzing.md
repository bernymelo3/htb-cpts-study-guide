# THEORY — Vhost Fuzzing

## ID
8

## Module
Ffuf

## Kind
theory

## Title
Section 8 — Vhost Fuzzing

## Description
Put FUZZ in the `Host:` header (not the URL) to discover vhosts that share an IP — even when no public DNS record exists.

## Tags
ffuf, vhost, host-header, lab, theory

## Commands
- `ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://academy.htb:PORT/ -H 'Host: FUZZ.academy.htb'`

## TL;DR — What's Important
- A **vhost** is another site served from the same IP, distinguished by the `Host:` request header.
- Vhost ≠ sub-domain: vhosts may have **no public DNS record** but still respond on the IP.
- Send the request **to the IP**; vary the `Host:` header value via FUZZ.
- Without filtering, every request returns the same default page (200 OK, same size). Section 9 (Filtering Results) closes this loop.

## Vhost vs Sub-domain
| Aspect | Sub-domain Fuzzing | Vhost Fuzzing |
|--------|--------------------|--------------|
| Where FUZZ goes | URL hostname | `Host:` header |
| Requires public DNS? | Yes | No |
| Discovers private vhosts? | No | Yes |
| Works on lab targets? | Rarely | Always |

## How it works
1. Connect to the IP directly — `-u http://academy.htb:PORT/` (resolves via `/etc/hosts`).
2. Override the `Host:` header per request: `-H 'Host: FUZZ.academy.htb'`.
3. Server's vhost router examines `Host:` and serves the matching site.
4. Existing vhost → different content → different response size/words/lines.
5. Non-existing vhost → server falls back to default vhost → identical baseline response.

## Why every result looks like 200 OK
The IP always answers; the question is whether the server has a vhost defined for the requested name. When no vhost matches, the server returns the **default vhost's** page, which means every fuzzing attempt gets 200 with identical body. You filter the noise by **response size** (`-fs`) — see section 9.

## Key Takeaways
- The `Host:` header is the deciding factor. The same wire-level connection serves wildly different sites depending on its value.
- Vhost fuzzing finds **dev/staging/admin** sub-sites that companies forgot to firewall off.
- Always vhost-fuzz lab boxes — public DNS won't reveal lab vhosts.

## Gotchas
- Forgetting `/etc/hosts` for the discovered vhost means your browser can't visit it after ffuf finds it. Add the entry, then visit.
- If the server has no default vhost, `Host:` mismatches may 404 instead of returning a baseline page — filtering logic flips.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ffuf]]
← [[07-sub-domain-fuzzing]] | [[09-filtering-results]] →
<!-- AUTO-LINKS-END -->
