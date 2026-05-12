# NOTE — Fingerprinting

## ID
210

## Module
Information Gathering – Web Edition

## Kind
lab

## Title
Section 11 — Fingerprinting

## Description
Identify web server, OS, CMS, and WAF by combining banner grabbing, header analysis, page-content inspection, and dedicated tools (`curl -I`, `wafw00f`, `nikto`, Wappalyzer, WhatWeb, BuiltWith, Netcraft).

## Tags
fingerprinting, banner-grabbing, headers, wafw00f, nikto, wappalyzer, whatweb, cms, waf

## Commands
- `curl -I http://inlanefreight.com`
- `curl -I https://inlanefreight.com`
- `curl -I https://www.inlanefreight.com`
- `curl -s http://app.inlanefreight.local/index.php | grep '<meta name="generator"'`
- `pip3 install git+https://github.com/EnableSecurity/wafw00f`
- `wafw00f inlanefreight.com`
- `nikto -h inlanefreight.com -Tuning b`
- `nikto -h http://dev.inlanefreight.local -Tuning b`

## What This Section Covers
Fingerprinting tells you what's running so you can target the right exploits. Server software + version, OS, CMS, frameworks, WAF — every detail narrows the attack surface. This section walks through manual (curl headers, page content) and automated (wafw00f, nikto, Wappalyzer) techniques against the lab targets.

## Methodology
1. **Banner grab with `curl -I`** — pulls just the response headers. Look for `Server`, `X-Powered-By`, `X-Redirect-By`, `Link`.
2. **Follow redirects** — initial domain often `301`s to `www.` or `https://`. Re-grab headers at each hop; new headers may appear (e.g. WordPress's `X-Redirect-By`).
3. **Inspect page content** for `<meta name="generator">`, copyright lines, JS framework references, distinctive paths (`/wp-json/`, `/_next/`, etc.).
4. **WAF check with `wafw00f`** — *before* further probing. WAF will affect every later step.
5. **Run `nikto -Tuning b`** — software-identification only (avoid the noisy full scan early on).
6. **Confirm with browser tools** — Wappalyzer extension auto-fingerprints CMS/frameworks/analytics.

## Manual Banner Grab — what you're looking for in headers
| Header | Tells you |
|---|---|
| `Server:` | Web server software + version (`Apache/2.4.41 (Ubuntu)` → Apache on Ubuntu) |
| `X-Powered-By:` | Backend tech (PHP version, ASP.NET, Express) |
| `X-Redirect-By:` | E.g. `WordPress` — confirms CMS even before page load |
| `Set-Cookie:` | Session cookie names hint at framework (`PHPSESSID`, `JSESSIONID`, `ASP.NET_SessionId`) |
| `Link: <.../wp-json/>;` | WordPress (REST API root) |
| `X-Generator:` | Often present for static-site generators |

## Tools Reference
| Tool | Use |
|---|---|
| **Wappalyzer** | Browser extension — passive fingerprint per page |
| **BuiltWith** | Web service — historical tech-stack profile |
| **WhatWeb** | CLI fingerprinter, large signature DB |
| **Nmap** (`-sV`, NSE scripts) | Service + version detection |
| **Netcraft** | Hosting + tech + uptime reports |
| **wafw00f** | WAF detection — Wordfence, Cloudflare, Akamai, etc. |
| **nikto** | Vuln scanner with `-Tuning b` for soft fingerprint mode |

## CMS Detection — the `<meta name="generator">` Trick
Most CMSes (Joomla, WordPress, Drupal, MediaWiki, etc.) emit a `<meta name="generator">` tag by default. Pull it with:
```bash
curl -s http://app.inlanefreight.local/index.php | grep '<meta name="generator"'
# <meta name="generator" content="Joomla! - Open Source Content Management" />
```
Wappalyzer also picks this up automatically when you browse the site.

## Nikto — Software Identification Mode
`-Tuning b` restricts Nikto to software-identification checks. Less noisy than a full scan, useful for early fingerprinting. Reveals:
- IPv4 + IPv6 addresses
- Server software/version
- Outdated software warnings
- CMS detection (`A Wordpress installation was found.`)
- Login pages (`/wp-login.php`)
- Information-disclosure files (`license.txt`)
- Missing security headers (HSTS, X-Content-Type-Options)

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Apache version on `app.inlanefreight.local` (format `0.0.0`) | **{see lab — answer redacted in walkthrough}** | `curl -I http://app.inlanefreight.local` → `Server:` header |
| Q2 — CMS used on `app.inlanefreight.local` | **{see lab — answer redacted in walkthrough}** | `curl -s http://app.inlanefreight.local/index.php \| grep '<meta name="generator"'` (or Wappalyzer browser extension) |
| Q3 — OS of `dev.inlanefreight.local` web server | **{see lab — answer redacted in walkthrough}** | `nikto -h http://dev.inlanefreight.local -Tuning b` → look at `Server:` line for distro hint (e.g. `Apache/2.4.41 (Ubuntu)`) |

## Key Takeaways
- Always start with `curl -I` — headers tell you a surprising amount with one packet.
- Run `wafw00f` before noisier tools — knowing there's a WAF changes your entire approach.
- `<meta name="generator">` is the lazy-but-effective CMS detector.
- Nikto's `-Tuning b` is the gentle fingerprint mode; full Nikto is way too noisy for early recon.

## Gotchas
- A `Server: nginx` header doesn't always mean nginx is the app server — it's often a reverse proxy in front of Apache/Node/whatever.
- WAFs sometimes inject misleading headers to confuse fingerprinting.
- A site behind a WAF may rate-limit Nikto into uselessness — slow it down with `-Pause` if needed.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← [[10-certificate-transparency-logs]] | [[12-crawling]] →
<!-- AUTO-LINKS-END -->
