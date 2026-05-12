# NOTE — Nmap Scripting Engine

## ID
7

## Module
Nmap

## Kind
notes

## Title
Section 7 — Nmap Scripting Engine (NSE)

## Description
NSE = Lua scripts that interact with services for auth, discovery, brute, vulnerability checks, and exploitation. Cover script categories, calling conventions, the aggressive (`-A`) shortcut, and `--script vuln` for CVE matching.

## Tags
nmap, nse, scripting, vuln, discovery, http-enum, banner

## Commands
- `sudo nmap <IP> -sC` — default scripts
- `sudo nmap <IP> --script <category>`
- `sudo nmap <IP> --script <name1>,<name2>,...`
- `sudo nmap <IP> -p 25 --script banner,smtp-commands`
- `sudo nmap <IP> -p 80 -A`
- `sudo nmap <IP> -p 80 -sV --script vuln`
- `sudo nmap -p80 <IP> --script discovery`
- `sudo nmap -p80 <IP> --script http-enum`

## Script Categories
| Category | Purpose |
|----------|---------|
| `auth` | Auth credential discovery |
| `broadcast` | Broadcast-based host discovery; auto-adds discovered hosts |
| `brute` | Login brute-force across services |
| `default` | Run with `-sC`; safe defaults |
| `discovery` | Service / topology enumeration |
| `dos` | DoS checks — destructive, rarely used |
| `exploit` | Real exploit attempts for known CVEs |
| `external` | Calls external services (e.g. WHOIS, Shodan) |
| `fuzzer` | Mutational input fuzzing |
| `intrusive` | Could disrupt the target |
| `malware` | Detect existing infections / backdoors |
| `safe` | Won't crash anything |
| `version` | Extends `-sV` detection |
| `vuln` | Tests for specific CVEs / weaknesses |

## How to Call Scripts
```bash
# Default scripts
sudo nmap <target> -sC

# A whole category
sudo nmap <target> --script vuln

# Specific scripts
sudo nmap <target> --script banner,smtp-commands
```

## Aggressive Scan (`-A`)
`-A` is a shortcut for `-sV -O -sC --traceroute`:
- Service version detection
- OS detection
- Default scripts
- Network traceroute

```bash
sudo nmap <IP> -p 80 -A
```
Loud but informative — gives you `http-generator: WordPress 5.3.4`, server header, page title, and an OS guess (e.g. `Linux 96%`).

## Vulnerability Sweep
```bash
sudo nmap <IP> -p 80 -sV --script vuln
```
Common scripts under `vuln`:
- `http-enum` → finds hidden directories like `/wp-login.php`, `/robots.txt`, `/wp-admin/`.
- `http-wordpress-users` → enumerates WP usernames.
- `vulners` → matches CPE strings against the Vulners DB and prints CVEs.
- `http-stored-xss`, `http-csrf` → web vuln spot checks.

The `vulners` output is gold — it lists real CVEs with CVSS scores and direct vulners.com links.

## Script Reference
- All scripts: https://nmap.org/nsedoc/index.html
- Local files: `/usr/share/nmap/scripts/*.nse`
- Search by name: `ls /usr/share/nmap/scripts/ | grep <keyword>`

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Flag in one of the services | **`HTB{873nniuc71bu6usbs1i96as6dsv26}`** | `--script discovery` (or just `http-enum`) found `/robots.txt`. `curl http://<IP>/robots.txt` returned the flag |

### Reference commands
```bash
sudo nmap -p80 <IP> --script discovery       # or --script http-enum (faster)
curl http://<IP>/robots.txt
```

## Key Takeaways
- `-sC` runs the *default* category — quick wins for free, no extra time cost on top of `-sV`.
- `--script vuln` + `vulners` gives you a CVE-mapped report straight from a port scan.
- `http-enum` regularly finds `/robots.txt`, admin panels, version-leaking files.
- Don't run `discovery` against many hosts at once — it's slow because it runs *every* discovery script.

## Gotchas
- `--script vuln` includes intrusive checks. Don't fire it at production without authorization.
- Some scripts are noisy enough to trigger IDS rules — e.g. `http-sql-injection`, brute scripts.
- `--script-args` exists for tuning; e.g. `http-enum.basepath=/admin/`.
- `-A` invokes `-O` which needs root and at least one open + one closed port to fingerprint reliably.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|nmap]]
← [[06-service-enumeration]] | [[08-performance]] →
<!-- AUTO-LINKS-END -->
