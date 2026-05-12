# THEORY — DNS Records & /etc/hosts

## ID
6

## Module
Ffuf

## Kind
theory

## Title
Section 6 — DNS Records

## Description
Why hitting `academy.htb` from a browser fails until you map the hostname → IP in `/etc/hosts`. Sets up the next two sections (sub-domain vs vhost fuzzing).

## Tags
ffuf, theory, dns, etc-hosts, hostname-resolution

## TL;DR — What's Important
- Browsers resolve hostnames via `/etc/hosts` first, then public DNS. HTB lab domains exist in **neither** by default.
- For lab `*.htb` domains, you must add the IP → hostname mapping yourself.
- This is critical because the **server's response can change based on the `Host` header** — the same IP can serve multiple sites (vhosts).

## Concept Overview
A pasted URL like `http://academy.htb` triggers a DNS lookup. Browsers don't know how to talk to "academy.htb" — they only speak IP. Resolution order:
1. Local `/etc/hosts` file
2. Configured DNS server (e.g. 8.8.8.8)

For HTB lab targets, neither of those knows about `academy.htb`. The resolver fails and the connection never starts. Adding the mapping locally lets your browser dial the right IP and send the `Host:` header — which is what triggers the vhost-aware response on the server.

## Adding /etc/hosts entries
```bash
sudo sh -c 'echo "SERVER_IP  academy.htb" >> /etc/hosts'
```

For multiple subdomains on the same IP:
```bash
sudo sh -c 'echo "SERVER_IP  academy.htb admin.academy.htb test.academy.htb" >> /etc/hosts'
```

## When this matters for fuzzing
| Scenario | Tool to use |
|----------|-------------|
| Public domain, want `*.example.com` discovery | Sub-domain fuzzing (DNS-based) |
| Lab IP, no public DNS for the domain | Vhost fuzzing (Host header) |
| Both | Vhost fuzzing — it works in both cases |

## Key Takeaways
- IP-direct browsing works without DNS but loses the `Host:` context — you'll see the default vhost only.
- Hostname browsing requires resolution + sends `Host:` so the server can route to the correct vhost.
- This is **the** reason `admin.academy.htb` shows different content from `academy.htb` even though both point to the same IP.

## Gotchas
- After IP changes (target restart), `/etc/hosts` becomes stale. Edit or delete the old line — don't append a duplicate.
- Don't forget the port when visiting (`http://academy.htb:31420`) — `/etc/hosts` has no concept of ports.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ffuf]]
← [[05-recursive-fuzzing]] | [[07-sub-domain-fuzzing]] →
<!-- AUTO-LINKS-END -->
