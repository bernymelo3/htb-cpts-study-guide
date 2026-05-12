# NOTE — Skills Assessment

## ID
218

## Module
Information Gathering – Web Edition

## Kind
lab

## Title
Section 19 — Skills Assessment

## Description
End-of-module skills assessment chaining WHOIS → fingerprinting → vhost brute-force → robots.txt analysis → recursive vhost discovery → crawling. Final answer (Q5) leaks the new API key from a crawler-discovered HTML comment.

## Tags
skills-assessment, lab, capstone, whois, gobuster-vhost, robots-txt, reconspider, full-chain

## Commands
- `whois inlanefreight.com | grep IANA`
- `sudo sh -c "echo '<STMIP> inlanefreight.htb' >> /etc/hosts"`
- `curl -I http://inlanefreight.htb:<PORT>`
- `gobuster vhost -u http://inlanefreight.htb:<PORT> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -t 60 --append-domain`
- `curl http://web1337.inlanefreight.htb:<PORT>/robots.txt`
- `curl -I http://web1337.inlanefreight.htb:<PORT>/admin_h1dd3n`
- `curl http://web1337.inlanefreight.htb:<PORT>/admin_h1dd3n/`
- `gobuster vhost -u http://web1337.inlanefreight.htb:<PORT> -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -t 60 --append-domain`
- `python3 ReconSpider.py http://dev.web1337.inlanefreight.htb:<PORT>`
- `cat results.json | jq '.emails'`
- `cat results.json | jq '.comments'`

## What This Section Covers
The capstone — chains every technique from this module:
1. WHOIS for an IANA ID.
2. `curl -I` for HTTP server software.
3. `gobuster vhost` round 1 → discovers `web1337.inlanefreight.htb`.
4. `robots.txt` on the new vhost reveals a `Disallow: /admin_h1dd3n` path → contains an API key in plaintext.
5. `gobuster vhost` round 2 against `web1337.inlanefreight.htb` → discovers nested `dev.web1337.inlanefreight.htb`.
6. ReconSpider crawl on the dev vhost → email + future-API-key in HTML comments.

## Full Attack Chain
```
inlanefreight.htb (root)
  ├─ whois → IANA ID
  ├─ curl -I → server software (nginx)
  └─ vhost brute-force →
       web1337.inlanefreight.htb
         ├─ /robots.txt → Disallow: /admin_h1dd3n
         ├─ /admin_h1dd3n/ → API key in HTML
         └─ vhost brute-force (against web1337.) →
              dev.web1337.inlanefreight.htb
                └─ ReconSpider crawl →
                     ├─ jq '.emails' → email
                     └─ jq '.comments' → "Remember to change the API key to ba988b835be4aa97d068941dc852ff33"
```

## Step-by-Step Methodology
### Step 1 — WHOIS IANA ID
```bash
whois inlanefreight.com | grep IANA
```

### Step 2 — Server fingerprint
After adding `inlanefreight.htb` to `/etc/hosts`:
```bash
curl -I http://inlanefreight.htb:<PORT>
```
Read the `Server:` header. (Walkthrough output shows `nginx/1.26.1`.)

### Step 3 — First-level vhost brute force
```bash
gobuster vhost -u http://inlanefreight.htb:<PORT> \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
  -t 60 --append-domain
```
Reveals `web1337.inlanefreight.htb`. Add to `/etc/hosts`.

### Step 4 — robots.txt on new vhost
```bash
curl http://web1337.inlanefreight.htb:<PORT>/robots.txt
```
Look for `Disallow:` — walkthrough exposes `/admin_h1dd3n`.

### Step 5 — Probe the disallowed path
```bash
curl -I http://web1337.inlanefreight.htb:<PORT>/admin_h1dd3n
# 301 → Location: /admin_h1dd3n/

curl http://web1337.inlanefreight.htb:<PORT>/admin_h1dd3n/
# HTML returns: "...the API is still accessible with the key <KEY>"
```

### Step 6 — Nested vhost brute force
```bash
gobuster vhost -u http://web1337.inlanefreight.htb:<PORT> \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
  -t 60 --append-domain
```
Finds `dev.web1337.inlanefreight.htb`. Add to `/etc/hosts`.

### Step 7 — Crawl the dev vhost
Install Scrapy + ReconSpider (see [[15-creepy-crawlies]]):
```bash
pip3 install scrapy --break-system-packages
wget https://academy.hackthebox.com/storage/modules/279/ReconSpider.zip ; unzip ReconSpider.zip
python3 ReconSpider.py http://dev.web1337.inlanefreight.htb:<PORT>
```
Then:
```bash
cat results.json | jq '.emails'    # Q4
cat results.json | jq '.comments'  # Q5 — leaks the new API key
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — IANA ID of registrar of `inlanefreight.com` | **{see lab — answer redacted in walkthrough}** | `whois inlanefreight.com \| grep IANA` |
| Q2 — HTTP server software powering `inlanefreight.htb` (name only, e.g. Apache) | **{see lab — answer redacted in walkthrough; walkthrough output shows `nginx`}** | `curl -I http://inlanefreight.htb:<PORT>` → `Server:` header |
| Q3 — API key in the hidden admin directory | **{see lab — answer redacted in walkthrough}** | `gobuster vhost` → `web1337.` → `/robots.txt` reveals `/admin_h1dd3n` → `curl /admin_h1dd3n/` returns key in HTML |
| Q4 — Email address found by crawling `inlanefreight.htb` | **{see lab — answer redacted in walkthrough}** | Recursive vhost brute → `dev.web1337.inlanefreight.htb` → `python3 ReconSpider.py http://dev.web1337.inlanefreight.htb:<PORT>` → `cat results.json \| jq '.emails'` |
| Q5 — API key the developers will be changing to | **`ba988b835be4aa97d068941dc852ff33`** | Same crawl → `cat results.json \| jq '.comments'` → `"Remember to change the API key to ba988b835be4aa97d068941dc852ff33"` |

## Key Takeaways
- After finding a vhost, **always re-run vhost brute-force against it** — this is how you found `dev.web1337.` nested under `web1337.`.
- **Always check `/robots.txt`** on every discovered vhost — the `Disallow:` lines pointed straight at `/admin_h1dd3n`.
- **Always crawl HTML comments** — devs leak credentials, future API keys, internal email addresses there. `jq '.comments'` is the fastest grep.
- Trailing slashes matter: `/admin_h1dd3n` returned 301 → `/admin_h1dd3n/`. The 301 redirect was a hint to follow.

## Gotchas
- The 110k wordlist takes much longer than the 20k — start with 20k for speed, escalate to 110k if needed.
- Modern Debian/Ubuntu/Pwnbox needs `pip3 install scrapy --break-system-packages` (PEP 668).
- Make sure every newly-found vhost is added to `/etc/hosts` *before* you try to hit it.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← [[18-automating-recon]] | *(end of module)*
<!-- AUTO-LINKS-END -->
