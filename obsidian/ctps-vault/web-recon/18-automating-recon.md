# NOTE — Automating Recon

## ID
217

## Module
Information Gathering – Web Edition

## Kind
theory

## Title
Section 18 — Automating Recon

## Description
Why and how to automate the recon stack: efficiency, scalability, consistency, comprehensive coverage, integration. Tour of the major frameworks (FinalRecon, Recon-ng, theHarvester, SpiderFoot, OSINT Framework) with a focused walkthrough of FinalRecon.

## Tags
automation, finalrecon, recon-ng, theharvester, spiderfoot, osint-framework, framework

## Commands
- `git clone https://github.com/thewhiteh4t/FinalRecon.git`
- `cd FinalRecon && pip3 install -r requirements.txt && chmod +x ./finalrecon.py`
- `./finalrecon.py --help`
- `./finalrecon.py --headers --whois --url http://inlanefreight.com`
- `./finalrecon.py --full --url http://inlanefreight.com`

## TL;DR — What's Important
- Automate to scale: many domains, repeatable, fewer human errors, integrates with downstream tools.
- Frameworks: FinalRecon (modular Python), Recon-ng (modular like Metasploit for OSINT), theHarvester (emails+subs), SpiderFoot (broad OSINT auto), OSINT Framework (catalog of resources).
- FinalRecon bundles headers + WHOIS + SSL + crawler + DNS + subdomains + dirs + Wayback + port-scan in one CLI.

## Why Automate
| Benefit | Detail |
|---|---|
| **Efficiency** | Repetitive lookups run unattended |
| **Scalability** | Same workflow against many targets |
| **Consistency** | Reproducible output, fewer human errors |
| **Coverage** | DNS + subs + ports + crawl + WHOIS in one shot |
| **Integration** | Output JSON/CSV → vuln scanner → exploit framework |

## Major Frameworks
| Tool | Strength |
|---|---|
| **FinalRecon** | Python, modular — single CLI for headers, SSL, WHOIS, crawler, DNS, subs, dirs, Wayback, port scan |
| **Recon-ng** | Modular like Metasploit; many OSINT modules + DB-backed workspace |
| **theHarvester** | Emails, subdomains, hosts, employees from search engines + PGP + Shodan |
| **SpiderFoot** | OSINT automation across many integrated data sources; web UI |
| **OSINT Framework** | Catalog/index of OSINT resources, not a tool itself |

## FinalRecon Modules (Flag Reference)
| Flag | Module |
|---|---|
| `--url URL` | Target URL (required) |
| `--headers` | Header info |
| `--sslinfo` | SSL/TLS cert info |
| `--whois` | WHOIS lookup |
| `--crawl` | Crawl target (HTML/CSS/JS, links, comments, files, robots.txt, sitemap.xml, Wayback) |
| `--dns` | DNS enumeration (A, AAAA, MX, NS, TXT, DMARC) |
| `--sub` | Subdomain enum (crt.sh, AnubisDB, ThreatMiner, CertSpotter, Facebook, VirusTotal, Shodan, BeVigil) |
| `--dir` | Directory search (custom wordlists + extensions) |
| `--wayback` | Wayback URLs (last 5 years) |
| `--ps` | Fast port scan |
| `--full` | Run everything |

### Tuning flags
| Flag | Purpose |
|---|---|
| `-dt N` | Threads for dir enum (default 30) |
| `-pt N` | Threads for port scan (default 50) |
| `-T N` | Request timeout (default 30s) |
| `-w PATH` | Custom wordlist for dir enum |
| `-r` | Allow redirects |
| `-s` | Toggle SSL verification |
| `-d IP` | Custom DNS server (default 1.1.1.1) |
| `-e txt,xml,php` | File extensions for dir enum |
| `-o txt\|json\|...` | Export format |
| `-cd PATH` | Change export directory |
| `-k svc@key` | Add API key |

## FinalRecon Install
```bash
git clone https://github.com/thewhiteh4t/FinalRecon.git
cd FinalRecon
pip3 install -r requirements.txt
chmod +x ./finalrecon.py
./finalrecon.py --help
```

## Example — headers + WHOIS only
```bash
./finalrecon.py --headers --whois --url http://inlanefreight.com
```
Returns IP, all response headers, full WHOIS — both modules in one invocation. Output is also saved to `~/.local/share/finalrecon/dumps/` for later review.

## Key Takeaways
- For one-off recon, individual tools (`dig`, `whois`, `wafw00f`, `gobuster`) give finer control.
- For repeatable / multi-target / time-boxed engagements, frameworks like FinalRecon save hours.
- Always store output for diffing — recon is most useful when you can compare snapshots over time.
- `--full` is convenient but noisy — pick targeted modules in real engagements to avoid blowing your cover with one command.

## Gotchas
- API-driven modules (Shodan, VirusTotal, BeVigil) need keys — without them, results are partial.
- Some modules duplicate work (`--crawl` already covers `--wayback`) — read help carefully to avoid redundant runs.
- Heavy modules (`--ps`, `--dir`) are very loud — get explicit authorization before running them.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← [[17-web-archives]] | [[19-skills-assessment]] →
<!-- AUTO-LINKS-END -->
