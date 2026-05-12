# NOTE — Virtual Hosts

## ID
208

## Module
Information Gathering – Web Edition

## Kind
lab

## Title
Section 9 — Virtual Hosts

## Description
Discover non-public vhosts hosted on a single IP by fuzzing the HTTP `Host` header. Covers vhost vs subdomain distinction, the three vhost types (name-based, IP-based, port-based), and `gobuster vhost` brute-force against `inlanefreight.htb`.

## Tags
vhost, virtual-host, host-header, gobuster, ffuf, feroxbuster, fuzzing, hosts-file

## Commands
- `sudo sh -c "echo '<STMIP> inlanefreight.htb' >> /etc/hosts"`
- `gobuster vhost -u http://inlanefreight.htb:<PORT>/ -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt --append-domain`
- `gobuster vhost -u http://inlanefreight.htb:<PORT>/ -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt --append-domain -t 60`
- `curl -H "Host: dev.inlanefreight.htb" http://<STMIP>:<PORT>/`

## What This Section Covers
A web server can host many sites on one IP using the HTTP `Host` header. **Subdomains have DNS records; vhosts don't have to.** That means a vhost can exist purely as a server config — invisible to DNS/CT logs, only discoverable by guessing its name and sending it as a `Host` header.

## Subdomain vs vhost (the key distinction)
| | Subdomain | Virtual Host |
|---|---|---|
| Has DNS record? | Yes | Not required |
| Discovery method | DNS enum, CT logs | `Host` header fuzzing |
| Config location | DNS server | Web server (Apache/Nginx/IIS) |
| Why it exists | Public-facing org | Often dev / internal / hidden |

If a vhost has no DNS record, you must add it to `/etc/hosts` (or use `curl -H "Host: ..."`) to reach it.

## How vhost Routing Works (server-side)
1. **Browser sends HTTP request** with `Host: app.example.com`.
2. **Web server reads `Host` header**.
3. **Server looks up vhost config** matching that hostname.
4. **Server serves files** from that vhost's DocumentRoot.

Apache example:
```apacheconf
<VirtualHost *:80>
    ServerName www.example1.com
    DocumentRoot /var/www/example1
</VirtualHost>
<VirtualHost *:80>
    ServerName www.example2.org
    DocumentRoot /var/www/example2
</VirtualHost>
```

## Three Types of Virtual Hosting
| Type | How it routes | Pros | Cons |
|---|---|---|---|
| **Name-based** | `Host` header | Cheap, scalable | TLS gets tricky (needs SNI) |
| **IP-based** | Destination IP | Works with any protocol | Needs multiple IPs |
| **Port-based** | Destination port | Works when IPs limited | Ugly URLs (`:8080`) |

## Vhost Discovery Tools
| Tool | Note |
|---|---|
| `gobuster vhost` | Fast, simple, the de-facto starter |
| `feroxbuster` | Rust-based, recursion + filters |
| `ffuf` | Most flexible — fuzz any header, custom matchers |

## Gobuster vhost Workflow
1. **Identify target IP and port**.
2. **Add the base domain to `/etc/hosts`** so DNS resolution works:
   ```bash
   sudo sh -c "echo '<STMIP> inlanefreight.htb' >> /etc/hosts"
   ```
3. **Run gobuster**:
   ```bash
   gobuster vhost -u http://inlanefreight.htb:<PORT>/ \
     -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
     --append-domain
   ```
4. **Examine output** — each `Found:` line is a discovered vhost.
5. **For each new vhost** — add to `/etc/hosts` and browse it.

## Useful Gobuster Flags
| Flag | Purpose |
|---|---|
| `-u` | Target URL |
| `-w` | Wordlist |
| `--append-domain` | Append base domain to each word (required in newer Gobuster) |
| `-t` | Threads (default 10; bump to 60 for speed) |
| `-k` | Ignore TLS cert errors |
| `-o` | Output file |

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Full subdomain prefixed with "web" | **{see lab — answer redacted in walkthrough}** | `gobuster vhost -u http://inlanefreight.htb:<PORT>/ -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt --append-domain` |
| Q2 — Full subdomain prefixed with "vm" | **{see lab — answer redacted in walkthrough}** | Same gobuster output |
| Q3 — Full subdomain prefixed with "br" | **{see lab — answer redacted in walkthrough}** | Same gobuster output |
| Q4 — Full subdomain prefixed with "a" | **{see lab — answer redacted in walkthrough}** | Same gobuster output |
| Q5 — Full subdomain prefixed with "su" | **{see lab — answer redacted in walkthrough}** | Same gobuster output |

## Key Takeaways
- Vhosts are *only* discoverable by guessing — no passive source will reveal them.
- Always add the base domain to `/etc/hosts` before running vhost tools, otherwise DNS will fail.
- `--append-domain` is required in newer gobuster versions; older versions did this implicitly.
- Vhost fuzzing is noisy — every guess is a real HTTP request to the target.

## Gotchas
- Wildcard vhosts: server returns the same default page for every Host header → all guesses look "valid". Filter on Content-Length or status code with `--exclude-length` / `-fs`.
- Some web servers return 200 with an empty body for unknown vhosts. Check the response size — gobuster prints `[Size: N]` for each find.
- TLS + SNI complications — for HTTPS, you may need `-k` and ensure `Host` header matches SNI.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← [[08-dns-zone-transfers]] | [[10-certificate-transparency-logs]] →
<!-- AUTO-LINKS-END -->
