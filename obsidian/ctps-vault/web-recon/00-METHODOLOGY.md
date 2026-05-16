# NOTE — Information Gathering / Web Reconnaissance Methodology (Exam Playbook)

## ID
723

## Module
Information Gathering – Web Edition

## Kind
methodology

## Title
Web Reconnaissance — Full External-Recon Methodology

## Description
End-to-end exam-ready playbook for external web recon: passive footprint (WHOIS → DNS → CT logs → dorking → Wayback) → active DNS enum (AXFR → subdomain brute) → virtual-host discovery → fingerprinting → crawling & content discovery → automation → hand-off downstream. Decision-tree first; every command drawn from this module's own notes.

## Tags
methodology, web-recon, recon, exam, cheatsheet, decision-tree, whois, dns, dig, axfr, zone-transfer, subdomain, vhost, gobuster, dnsenum, crt.sh, ct-logs, fingerprinting, wafw00f, nikto, robots.txt, well-known, crawling, reconspider, google-dork, wayback, finalrecon, no-subdomain, hidden-vhost, stuck-recon

---

## TL;DR — The 7-Phase Flow

1. **Passive footprint** — WHOIS, DNS records, CT logs, search dorking, Wayback. Zero packets to target.
2. **Active DNS enum** — try AXFR first (one query = whole zone), then subdomain brute-force.
3. **Virtual-host discovery** — fuzz the `Host` header for sites with no DNS record.
4. **Fingerprinting** — server / OS / CMS / WAF on every live host. WAF check *before* noisy tools.
5. **Crawling & content discovery** — `robots.txt`, `/.well-known/`, `sitemap.xml`, ReconSpider; mine HTML comments.
6. **Automation** — FinalRecon to repeat the whole stack across many targets / time-boxed runs.
7. **Hand-off** — feed vhost list → ffuf, stack → web-attacks, internal IPs (from AXFR) → pivoting.

> **Golden rule:** passive before active — it's free and the target sees nothing, so exhaust it first. AXFR before brute-force — one successful zone transfer skips subdomain brute entirely. And: after every new vhost, **re-run vhost brute against it** and **check its `/robots.txt` + crawl its comments** — nested vhosts and leaked keys hide one level down (this is the entire Skills Assessment chain).

> **OPSEC fork:** active recon tips your hand. Brute-force lands in the authoritative-NS logs; `nikto`/`gobuster`/`FinalRecon --full` are every-guess-is-a-real-request loud. **Don't fire the loud tools until scope is confirmed.** Passive sources (crt.sh, dorking, Wayback, WHOIS) give 80% of the map for 0% of the noise — start there every time.

---

## Phase 1 — Passive Footprint (no traffic to target infra)

**Goal:** map the external surface before the target can see a single packet.

| Data to collect | Trigger / Precondition | Where / how |
|---|---|---|
| Registrar, IANA ID, contacts, NS, dates | Have a domain name | `whois <DOMAIN>` + `grep` |
| A / AAAA / MX / NS / TXT / SOA / PTR | Have a domain name | `dig` per-type |
| Subdomains (historical, incl. dead) | Domain has ever had a public TLS cert | `crt.sh` JSON API |
| Indexed files / login portals / configs | Domain is public | Google/Bing/DDG dorks |
| Old endpoints, dead subs, removed content | Site has history | Wayback Machine |

```bash
# WHOIS — install if missing, then filter the field you want
sudo apt update && sudo apt install whois -y
whois <DOMAIN> | grep IANA                  # registrar IANA ID
whois <DOMAIN> | grep "admin"               # admin contact
whois <DOMAIN> | grep -i "name server"      # NS hint at hosting

# DNS — never query only A
dig <DOMAIN>                                # default A (verbose)
dig +short <DOMAIN>                         # script-friendly
dig +noall +answer <DOMAIN>                 # answer section only
dig <DOMAIN> MX                             # mail infra
dig <DOMAIN> NS                             # authoritative NS (need this for Phase 2 AXFR)
dig <DOMAIN> TXT                            # SaaS leaks: SPF/DKIM/_verification
dig <DOMAIN> SOA
dig -x <IP>                                 # reverse PTR (auto-builds in-addr.arpa)
dig @1.1.1.1 <DOMAIN>                       # specific resolver (public vs internal view)
dig +trace <DOMAIN>                         # full root→TLD→authoritative walk

# CT logs — every issued cert leaks its SAN list (stealth subdomain dump)
curl -s "https://crt.sh/?q=<DOMAIN>&output=json" | jq -r '.[] | .name_value' | sort -u
curl -s "https://crt.sh/?q=<DOMAIN>&output=json" | jq -r '.[] | select(.name_value | contains("dev")) | .name_value' | sort -u
```

Search-engine dorks (run in a browser — zero packets to target):

```
site:<DOMAIN>                              # scope to one domain
site:<DOMAIN> (inurl:login OR inurl:admin) # portals
site:<DOMAIN> filetype:pdf                 # exposed docs
site:<DOMAIN> (filetype:xls OR filetype:docx)
site:<DOMAIN> inurl:config.php             # config files
site:<DOMAIN> (ext:conf OR ext:cnf)
site:<DOMAIN> inurl:backup
site:<DOMAIN> filetype:sql                 # DB backups
```
Pre-built dorks: Google Hacking Database (exploit-db.com/google-hacking-database). Try Bing + DuckDuckGo too — different indexes. Check `cache:<DOMAIN>` for content since removed.

Wayback: `https://web.archive.org/` → enter URL → pick a date → browse. Old `/robots.txt` & `/sitemap.xml` snapshots often list paths the current ones hide; deleted endpoints frequently still resolve live. Automate with `waybackurls` / `gau` / `cariddi`.

**Output checkpoint — after this you have:** registrar/IANA ID + contacts, full DNS record set, authoritative NS list (feeds Phase 2 AXFR), a passive subdomain list (crt.sh ∪ dorks ∪ Wayback), and any indexed sensitive files.

Detail: `[[02-whois]]`, `[[03-utilizing-whois]]`, `[[04-dns]]`, `[[05-digging-dns]]`, `[[10-certificate-transparency-logs]]`, `[[16-search-engine-discovery]]`, `[[17-web-archives]]`.

---

## Phase 2 — Active DNS Enumeration

**You have:** a domain + its authoritative NS list (from Phase 1 `dig NS`).
**Goal:** enumerate live subdomains. **Always try AXFR first** — if it works you skip brute-force entirely.

### 2.A — Zone Transfer (cheapest, highest payoff)

**Trigger:** you have the domain's NS records and at least one NS is reachable on 53/tcp.

```bash
dig NS <DOMAIN>                                              # 1. get every NS
dig axfr @<NS_HOST_OR_IP> <DOMAIN>                           # 2. try AXFR against EACH NS
dig axfr @nsztm1.digi.ninja zonetransfer.me                  # known-good public test target
dig axfr <DOMAIN> @<NS_IP> | grep "ftp.admin.<DOMAIN>"       # grep a specific host
dig axfr <DOMAIN> @<NS_IP> | grep "10.10.200"                # harvest internal IPs
```
Open AXFR dumps every subdomain, internal RFC1918 IP, MX/NS, and service hostname in one query. Record the internal IP ranges — they're next-hop targets for pivoting later.

### 2.B — Subdomain Brute-Force (when AXFR denied)

**Trigger:** AXFR failed/partial on every NS; passive list feels incomplete.

```bash
dnsenum --enum <DOMAIN> -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
dnsenum --enum <DOMAIN> -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt -r   # -r = recurse subs-of-subs (slow)
```
Other tools (interchangeable): `fierce` (wildcard detection), `dnsrecon`, `amass` (heavy passive sources), `assetfinder`, `puredns` (fast + wildcard filtering). Wordlist quality > tool choice — start with SecLists `subdomains-top1million-20000.txt`, escalate to `-110000` only if needed.

**Output checkpoint — after this you have:** the union of passive + active subdomains (live + dead), plus — if AXFR worked — internal IP ranges for the pivoting phase.

Detail: `[[06-subdomains]]`, `[[07-subdomain-bruteforcing]]`, `[[08-dns-zone-transfers]]`.

---

## Phase 3 — Virtual-Host Discovery

**You have:** an IP serving HTTP(S) and/or a base domain.
**Trigger:** suspect sites with **no DNS record** (dev/internal/hidden) co-hosted on one IP — nothing in DNS or CT logs will ever reveal these; the only way is to guess the name and send it as a `Host` header.

```bash
# 1. base domain MUST be in /etc/hosts first or DNS resolution fails
sudo sh -c "echo '<TARGET_IP> <DOMAIN>' >> /etc/hosts"

# 2. fuzz the Host header
gobuster vhost -u http://<DOMAIN>:<PORT>/ \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  --append-domain -t 60

# escalate wordlist if nothing found
gobuster vhost -u http://<DOMAIN>:<PORT>/ \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
  --append-domain -t 60

# 3. manual confirm a single guess
curl -H "Host: dev.<DOMAIN>" http://<TARGET_IP>:<PORT>/
```
For each `Found:` line → add the vhost to `/etc/hosts`, browse it, **then re-run vhost brute *against that vhost*** (nested vhosts live one level down). Alternatives: `feroxbuster`, `ffuf` (most flexible — custom matchers for wildcard filtering).

**Output checkpoint — after this you have:** every reachable vhost added to `/etc/hosts`, each one recursively brute-forced for nested vhosts.

Detail: `[[09-virtual-hosts]]`.

---

## Phase 4 — Fingerprinting

**You have:** one or more reachable web hosts/vhosts.
**Goal:** identify server/OS/CMS/WAF so you target the right exploits downstream. **WAF check before any noisy tool** — it changes the whole approach.

```bash
curl -I http://<HOST>                                  # banner: Server, X-Powered-By, X-Redirect-By, Link, Set-Cookie
curl -I https://www.<HOST>                             # re-grab at EACH redirect hop — new headers appear (e.g. X-Redirect-By: WordPress)
curl -s http://<HOST>/index.php | grep '<meta name="generator"'   # CMS tell (Joomla/WP/Drupal)

pip3 install git+https://github.com/EnableSecurity/wafw00f
wafw00f <HOST>                                         # run BEFORE nikto/gobuster

nikto -h <HOST> -Tuning b                              # -Tuning b = software-ID only (gentle); full nikto is far too loud early
```
Header tells: `Server:` → web server + OS distro; `X-Powered-By:` → backend; `Set-Cookie:` → framework (`PHPSESSID`/`JSESSIONID`/`ASP.NET_SessionId`); `Link: .../wp-json/` → WordPress. Confirm with the Wappalyzer/WhatWeb/BuiltWith/Netcraft layer.

**Output checkpoint — after this you have:** server software+version, OS, CMS+version, and WAF presence per host → feeds `attacking-common-applications/` and `web-attacks/`.

Detail: `[[11-fingerprinting]]`.

---

## Phase 5 — Crawling & Content Discovery

**You have:** a reachable web app.
**Goal:** map linked resources and mine the byproducts (comments, metadata, sensitive files). Crawling follows existing links; **fuzzing guesses unlinked paths — do both.**

```bash
# Free intel first — always, on EVERY vhost
curl http://<HOST>/robots.txt           # every Disallow: line points at something hidden
curl http://<HOST>/sitemap.xml          # structured URL list — skips crawling

# .well-known metadata (RFC 8615)
curl http://<HOST>/.well-known/security.txt
curl http://<HOST>/.well-known/openid-configuration   # OIDC goldmine: every endpoint + signing alg
curl http://<HOST>/.well-known/change-password
curl http://<HOST>/.well-known/mta-sts.txt

# Structured crawl with ReconSpider
pip3 install scrapy --break-system-packages            # PEP 668 on modern Debian/Pwnbox
wget -O ReconSpider.zip https://academy.hackthebox.com/storage/modules/144/ReconSpider.v1.2.zip
unzip ReconSpider.zip
python3 ReconSpider.py http://<HOST>
cat results.json | jq '.comments'                      # #1 highest-signal — TODOs, creds, future API keys
cat results.json | jq '.emails'
cat results.json | jq '.links'
cat results.json | jq '.js_files'                      # forgotten endpoints / 3rd-party svcs
```
`Disallow:` paths → add to your fuzzing wordlist (don't blindly hammer them — often instrumented as honeypots). HTML comments routinely leak credentials and future API keys (the Skills Assessment Q5 answer is literally `"Remember to change the API key to ..."` in a comment). Hunt for sensitive extensions in the link list: `.bak`, `.old`, `.swp`, `.git`, `.env`, `web.config`, `settings.php`, `error_log`.

**Output checkpoint — after this you have:** site map, robots/sitemap paths, OIDC attack surface (if any), and harvested comments/emails/JS endpoints.

Detail: `[[12-crawling]]`, `[[13-robots-txt]]`, `[[14-well-known-uris]]`, `[[15-creepy-crawlies]]`.

---

## Phase 6 — Automation (multi-target / time-boxed)

**Trigger:** many domains, or a repeatable/time-boxed engagement where running every tool by hand is too slow.

```bash
git clone https://github.com/thewhiteh4t/FinalRecon.git
cd FinalRecon && pip3 install -r requirements.txt && chmod +x ./finalrecon.py
./finalrecon.py --headers --whois --url http://<HOST>          # targeted, quiet
./finalrecon.py --full --url http://<HOST>                     # everything — LOUD, scope-confirmed only
```
Module flags: `--headers --sslinfo --whois --crawl --dns --sub --dir --wayback --ps`. Output saved to `~/.local/share/finalrecon/dumps/` — keep it for diffing snapshots over time. For one-off recon, individual tools (`dig`, `whois`, `wafw00f`, `gobuster`) give finer control and less noise. Alternatives: Recon-ng, theHarvester (emails+subs), SpiderFoot, OSINT Framework.

Detail: `[[18-automating-recon]]`.

---

## Phase 7 — Hand-off (recon feeds the rest of CPTS)

| Output | Goes to |
|---|---|
| Vhost / subdomain list | `ffuf` directory & param fuzzing → `../ffuf/` |
| Fingerprinted stack (CMS/app + version) | `../attacking-common-applications/00-METHODOLOGY.md`, `../web-attacks/00-METHODOLOGY.md` |
| Login portals (dorked / crawled) | `../login-brute-forcing/00-METHODOLOGY.md` |
| Internal IP ranges (from open AXFR) | `../pivoting-tunneling/00-METHODOLOGY.md` |
| Leaked creds / API keys (comments, dorks) | spray across services → `../password-attacks/00-METHODOLOGY.md` |
| `?file=` / `?page=` params (from crawl) | `../file-inclusion/02-lfi-basics.md` |

---

## Decision Tree (Under Exam Pressure)

```
You have:
│
├── only a domain name
│   ├── whois | grep IANA / admin / name server
│   ├── dig NS/MX/TXT/A  (TXT = SaaS leaks; NS = AXFR targets)
│   ├── crt.sh JSON  → passive subdomain list (free, stealth)
│   ├── site: dorks + Wayback  → indexed files / dead endpoints
│   └── STUCK > 10 min → move to active DNS (Phase 2)
│
├── domain + its NS records
│   ├── dig axfr @<each NS> <DOMAIN>  ← ALWAYS try first
│   │   └── works → whole zone (subs + internal IPs) in one shot, SKIP brute
│   └── denied/partial on all NS → dnsenum -f subdomains-top1million-20000.txt
│       └── STUCK / nothing → escalate -110000 list, or it's vhost-only → Phase 3
│
├── an IP serving HTTP, suspect hidden sites
│   ├── echo '<IP> <DOMAIN>' >> /etc/hosts   (MUST do first)
│   ├── gobuster vhost --append-domain (20k → 110k)
│   ├── each Found: → add to /etc/hosts → browse → RE-RUN vhost brute on it
│   └── all guesses "found" w/ same size → wildcard vhost → filter -fs / --exclude-length
│
├── a reachable web host
│   ├── wafw00f FIRST  → know the WAF before going loud
│   ├── curl -I (every redirect hop) + grep '<meta generator>'
│   ├── nikto -Tuning b  (gentle software-ID only)
│   └── then crawl: robots.txt → sitemap.xml → /.well-known/ → ReconSpider
│
├── a discovered vhost / new web root
│   ├── curl /robots.txt   → Disallow: lines = where to look
│   ├── ReconSpider → jq '.comments'  → leaked keys/emails/TODOs
│   ├── re-run vhost brute against THIS vhost  → nested vhost
│   └── /.well-known/openid-configuration → full OIDC surface
│
└── STUCK > 20 min
    ├── grep ../ATTACK-PATHS.md for current symptom
    ├── Did you try AXFR on EVERY NS? (only one is often misconfigured)
    ├── Did you re-brute every vhost for nested ones? (Skills Assessment trap)
    ├── Did you jq EVERY results.json key, not just .comments?
    ├── Wildcard DNS/vhost faking hits? → validate via HTTP fetch / size diff
    └── Check Wayback /robots.txt + /sitemap.xml snapshots for hidden paths
```

---

## Signal → Counter-Move Reference

| You see / observe | Likely cause | Counter-move |
|---|---|---|
| Every subdomain guess "resolves" | Wildcard DNS (`*.dom → 1.2.3.4`) | Validate with HTTP fetch; compare resolved IP against the wildcard IP; use `puredns`/`fierce` wildcard filtering |
| Every vhost guess "Found" same `[Size: N]` | Wildcard vhost (default page for any Host) | Filter on Content-Length: `ffuf -fs <N>` or gobuster `--exclude-length` |
| `dig axfr` returns only the SOA record | AXFR restricted (partial = deny, not success) | Try AXFR against **every** NS — often only one is misconfigured |
| `dig axfr` "no records / Transfer failed" | Denied **or** NS down | Confirm reachability: plain `dig <DOMAIN> @<NS>` first; AXFR is 53/**tcp** |
| `dig <DOMAIN> ANY` returns almost nothing | ANY filtered (RFC 8482) | Fall back to per-type queries (A/MX/NS/TXT/SOA individually) |
| `crt.sh` output is mostly `*.<DOMAIN>` | Wildcard cert collapses the SAN list | CT logs blind here — supplement with subdomain brute-force |
| `Server: nginx` but app behaves like PHP/Apache | nginx is a reverse proxy, not the app server | Don't trust one header; corroborate with `Set-Cookie`/`X-Powered-By`/page content |
| Fingerprint tools disagree / weird headers | WAF injecting misleading headers | Run `wafw00f` first; trust behaviour over a single header |
| `nikto` crawls to a halt / garbage results | WAF rate-limiting it | Slow down (`-Pause`), or rely on `curl -I` + Wappalyzer |
| `gobuster vhost` 200s with empty bodies everywhere | Unknown-vhost default response | Check `[Size: N]`; ensure `--append-domain` is set (required in new gobuster) |
| Can't reach a discovered vhost at all | Not in `/etc/hosts` | `sudo sh -c "echo '<IP> <VHOST>' >> /etc/hosts"` **before** hitting it |
| ReconSpider misses obvious links | JS-heavy SPA — links rendered client-side | Use a headless-browser crawler (Playwright/Puppeteer) |
| `pip3 install scrapy` → "externally-managed-environment" | PEP 668 (modern Debian/Ubuntu/Pwnbox) | `pip3 install scrapy --break-system-packages` |
| `dnsenum` stalls / Google module errors | Google-scraping rate-limited | Skip the scrape; brute module still runs — or `--noreverse` |
| SecLists wordlist path "not found" | Distro path differs | Check actual path (`/usr/share/seclists` on Pwnbox/Kali; varies elsewhere) |
| Found a vhost but no further progress | Didn't recurse | Re-run vhost brute *against the new vhost* + pull its `/robots.txt` |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# === Scenario 1: Cold start, only a domain ===
whois <DOMAIN> | grep -iE "IANA|admin|name server"
dig +short <DOMAIN>; dig <DOMAIN> NS +short; dig <DOMAIN> MX +short; dig <DOMAIN> TXT
curl -s "https://crt.sh/?q=<DOMAIN>&output=json" | jq -r '.[].name_value' | sort -u

# === Scenario 2: Zone transfer sweep (try EVERY NS) ===
for ns in $(dig +short NS <DOMAIN>); do echo "== $ns =="; dig axfr @"$ns" <DOMAIN>; done
dig axfr <DOMAIN> @<NS_IP> | grep -E "10\.|192\.168\.|172\.(1[6-9]|2[0-9]|3[01])\."   # internal IPs

# === Scenario 3: Subdomain brute (AXFR denied) ===
dnsenum --enum <DOMAIN> -f /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt

# === Scenario 4: Vhost discovery (hidden, no DNS) ===
sudo sh -c "echo '<TARGET_IP> <DOMAIN>' >> /etc/hosts"
gobuster vhost -u http://<DOMAIN>:<PORT>/ -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt --append-domain -t 60
curl -H "Host: dev.<DOMAIN>" http://<TARGET_IP>:<PORT>/

# === Scenario 5: Fingerprint (WAF first!) ===
wafw00f <HOST>
curl -I http://<HOST>; curl -I https://www.<HOST>
curl -s http://<HOST>/index.php | grep '<meta name="generator"'
nikto -h <HOST> -Tuning b

# === Scenario 6: Content discovery on a vhost ===
curl http://<HOST>/robots.txt; curl http://<HOST>/sitemap.xml
curl http://<HOST>/.well-known/openid-configuration
python3 ReconSpider.py http://<HOST>
cat results.json | jq '.comments, .emails, .js_files'

# === Scenario 7: Skills-Assessment chain (the recursive trap) ===
whois <DOMAIN> | grep IANA
curl -I http://<DOMAIN>:<PORT>                                   # Server: header
gobuster vhost -u http://<DOMAIN>:<PORT> -w .../subdomains-top1million-110000.txt -t 60 --append-domain
# → found web1337.<DOMAIN>: add to /etc/hosts
curl http://web1337.<DOMAIN>:<PORT>/robots.txt                   # Disallow: /admin_h1dd3n
curl http://web1337.<DOMAIN>:<PORT>/admin_h1dd3n/                # API key in HTML
gobuster vhost -u http://web1337.<DOMAIN>:<PORT> -w .../subdomains-top1million-110000.txt -t 60 --append-domain
# → found dev.web1337.<DOMAIN>: add to /etc/hosts
python3 ReconSpider.py http://dev.web1337.<DOMAIN>:<PORT>
cat results.json | jq '.emails'; cat results.json | jq '.comments'   # leaked future API key

# === Scenario 8: Passive-only (stealth, scope unconfirmed) ===
curl -s "https://crt.sh/?q=<DOMAIN>&output=json" | jq -r '.[].name_value' | sort -u
# browser: site:<DOMAIN> (inurl:login OR inurl:admin) ; site:<DOMAIN> filetype:pdf
# browser: https://web.archive.org/  → old /robots.txt + /sitemap.xml snapshots

# === Scenario 9: Automate the whole stack across targets ===
./finalrecon.py --headers --whois --sub --dns --crawl --url http://<HOST>
```

---

## Quick Reference — Tools by Function

| Function | Tool(s) | Note |
|---|---|---|
| Registration data | `whois` (+ `grep`) | passive; pipe through grep for the one field |
| DNS records | `dig` (+`+short`/`+trace`/`-x`/`@server`), `nslookup`, `host` | `dig` does every type; ANY is RFC-8482-filtered |
| Zone transfer | `dig axfr @<ns> <dom>`, `dnsenum` | try **every** NS; 53/tcp |
| Subdomain brute | `dnsenum`, `fierce`, `dnsrecon`, `amass`, `assetfinder`, `puredns` | wordlist quality > tool; SecLists top-20k → 110k |
| Passive subdomains | `crt.sh` JSON, Censys, SecurityTrails, VirusTotal, AnubisDB | finds dead/expired subs too |
| Vhost fuzzing | `gobuster vhost`, `feroxbuster`, `ffuf` | `/etc/hosts` first; `--append-domain` required |
| Fingerprint | `curl -I`, `wafw00f`, `nikto -Tuning b`, Wappalyzer, WhatWeb, BuiltWith, Netcraft, Nmap `-sV` | wafw00f BEFORE noisy tools |
| Crawl | ReconSpider (Scrapy), Burp Spider, OWASP ZAP, Apache Nutch | `jq '.comments'` = #1 signal |
| Content discovery | `robots.txt`, `sitemap.xml`, `/.well-known/*` | free intel — every vhost |
| Search OSINT | Google/Bing/DDG operators, GHDB | `site:` + one other operator |
| Historical | Wayback Machine, `waybackurls`, `gau`, `cariddi` | dead endpoints often still resolve |
| Automation | FinalRecon, Recon-ng, theHarvester, SpiderFoot | `--full` is loud — targeted flags in real engagements |

---

## Top Gotchas (Things That Will Burn You)

1. **AXFR before brute-force.** One open zone transfer hands you every subdomain *and* internal IPs in a single query — don't grind a 110k wordlist before trying `dig axfr` against every NS.
2. **Try AXFR on *every* NS.** Frequently only one secondary is misconfigured. A partial transfer (SOA only) is a **deny**, not a success.
3. **AXFR uses 53/tcp**, not UDP. "No records" can mean the NS is unreachable — confirm with a plain `dig <DOMAIN> @<NS>` first.
4. **Vhosts are invisible to passive recon.** No DNS record, not in CT logs — the *only* way to find them is `Host`-header fuzzing. Don't conclude "no subdomains" from crt.sh/dig alone.
5. **`/etc/hosts` before every vhost.** Add the base domain before running `gobuster vhost`, and add each discovered vhost before browsing it — otherwise DNS resolution fails and you'll think the host is dead.
6. **`--append-domain` is mandatory** in newer gobuster (older versions did it implicitly). Forget it and every result is wrong.
7. **The recursion trap (Skills Assessment).** After finding a vhost you must **re-run vhost brute against it** — nested vhosts (`dev.web1337.`) only appear when you brute the parent vhost. This is the single most common reason people stall on the capstone.
8. **Always pull `/robots.txt` on *every* vhost.** Its `Disallow:` lines point straight at hidden admin paths (the Skills Assessment API key sat behind `Disallow: /admin_h1dd3n`).
9. **`jq` every `results.json` key, not just `.comments`.** Emails, `js_files`, `external_files` leak third-party services and forgotten endpoints. Comments are #1 but not the only goldmine.
10. **Wildcard DNS / wildcard vhost fakes every hit.** Everything "resolves" / every guess "Found". Validate with an actual HTTP fetch and filter on response size (`-fs` / `--exclude-length`).
11. **`wafw00f` BEFORE nikto/gobuster.** A WAF changes your entire approach and will rate-limit loud tools into uselessness; knowing it's there first saves the engagement.
12. **`Server:` is often a reverse proxy** (nginx in front of Apache/Node). Don't pick exploits off one header — corroborate with cookies, `X-Powered-By`, and page content.
13. **`dig <DOMAIN> ANY` is RFC-8482-filtered** — returns almost nothing on modern resolvers. Always fall back to per-type queries.
14. **Wildcard certs collapse the crt.sh SAN list.** When CT logs are all `*.<DOMAIN>`, passive subdomain discovery is blind there — switch to brute-force.
15. **`pip3 install scrapy` needs `--break-system-packages`** on modern Debian/Ubuntu/Pwnbox (PEP 668). Without it the install silently fails and ReconSpider won't run.
16. **`nikto -Tuning b` is the gentle mode** (software-ID only). Full nikto is far too loud for early recon and gets you rate-limited/blocked.
17. **Don't blindly hammer every `Disallow:` path** on a live target — they're often instrumented honeypots; the first hit can flag your IP.
18. **`FinalRecon --full` / `--ps` / `--dir` are very loud.** Get authorization; in real engagements pick targeted modules. Convenient ≠ stealthy.
19. **Passive recon is free — actually do it.** crt.sh, dorking, and Wayback give most of the map with zero packets to the target. Skipping straight to active brute-force is both noisy and slower.
20. **Wayback endpoints often still resolve live.** Pages deleted from nav aren't deleted from disk — test archived URLs against the live site.

---

## Related Vault Notes

- `[[01-introduction]]` — active vs passive recon split
- `[[02-whois]]` / `[[03-utilizing-whois]]` — registration intel, IANA ID, contacts
- `[[04-dns]]` / `[[05-digging-dns]]` — record types, `dig` modifiers, `/etc/hosts`
- `[[06-subdomains]]` — active vs passive enumeration trade-off
- `[[07-subdomain-bruteforcing]]` — dnsenum + tool comparison
- `[[08-dns-zone-transfers]]` — AXFR (cheapest highest-payoff DNS recon)
- `[[09-virtual-hosts]]` — Host-header fuzzing, gobuster vhost
- `[[10-certificate-transparency-logs]]` — crt.sh JSON API
- `[[11-fingerprinting]]` — curl headers, wafw00f, nikto, CMS meta tag
- `[[12-crawling]]` — crawl vs fuzz, what to extract
- `[[13-robots-txt]]` — Disallow as a treasure map
- `[[14-well-known-uris]]` — security.txt, openid-configuration (OIDC surface)
- `[[15-creepy-crawlies]]` — ReconSpider + jq
- `[[16-search-engine-discovery]]` — Google dork operators + GHDB
- `[[17-web-archives]]` — Wayback Machine
- `[[18-automating-recon]]` — FinalRecon module flags
- `[[19-skills-assessment]]` — full chain: whois → fingerprint → recursive vhost → robots.txt → crawl

External cross-vault:
- Next: directory/param fuzzing → `[[../ffuf/00-overview]]` (when added)
- Stack identified → `[[../attacking-common-applications/00-METHODOLOGY]]`, `[[../web-attacks/00-METHODOLOGY]]`
- TXT-record / domain OSINT overlap → `[[../footprinting/03-domain-information]]`
- Internal IPs from AXFR → `[[../pivoting-tunneling/00-METHODOLOGY]]`
- Triage by symptom → `[[../ATTACK-PATHS]]` §3 (Web) / §9 (recon gaps)
- Index → `[[../SEARCH]]`
