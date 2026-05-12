# Section 3 — WordPress - Discovery & Enumeration

## ID
531

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 3 — WordPress - Discovery & Enumeration

## Description
Covers manual and automated WordPress enumeration using cURL, page source analysis, and WPScan to identify versions, themes, plugins, users, and vulnerabilities.

## Tags
wordpress, wpscan, enumeration, cms, plugins, web

## Commands
- `curl -s http://<TARGET> | grep WordPress`
- `curl -s http://<TARGET>/ | grep themes`
- `curl -s http://<TARGET>/ | grep plugins`
- `wpscan --url http://<TARGET> --enumerate`
- `wpscan --url http://<TARGET> --enumerate --api-token <API_TOKEN>`
- `wpscan --url http://<TARGET> --enumerate ap`
- `sudo gem install wpscan`

## What This Section Covers
How to discover and enumerate WordPress installations, including identifying the core version, installed themes and plugins (with version numbers), valid usernames, and known vulnerabilities. Combines manual techniques (cURL grepping, browsing page source, checking directory listings) with automated scanning via WPScan. WordPress powers ~32.5% of all sites on the internet, making it one of the most common targets during external pentests.

## Methodology

1. **Confirm WordPress is in use** — browse to `/robots.txt` (look for `/wp-admin/`, `/wp-content/`), try `/wp-login.php`, or grep page source:
   ```
   curl -s http://blog.inlanefreight.local | grep WordPress
   ```
   The `<meta name="generator">` tag often reveals the exact version.

2. **Identify the theme** by grepping for `themes` in the page source:
   ```
   curl -s http://blog.inlanefreight.local/ | grep themes
   ```

3. **Enumerate plugins** by grepping for `plugins` across multiple pages:
   ```
   curl -s http://blog.inlanefreight.local/ | grep plugins
   curl -s http://blog.inlanefreight.local/?p=1 | grep plugins
   ```
   Browse different pages — some plugins only load on specific page types (e.g., wpDiscuz only on posts with comments).

4. **Fingerprint plugin versions** — browse to `/wp-content/plugins/<plugin-name>/` and check for `readme.txt`. The `Stable tag` field in the readme gives the version.

5. **Enumerate users** — attempt login at `/wp-login.php` with known/guessed usernames:
   - Valid username + wrong password → "The password for username **admin** is incorrect"
   - Invalid username → "The username **someone** is not registered on this site"
   
   This differential response enables username enumeration.

6. **Check for directory listing** — browse `/wp-content/uploads/` and plugin/theme directories. Directory listing can expose flags, backups, or sensitive files.

7. **Run WPScan** for automated validation:
   ```
   wpscan --url http://blog.inlanefreight.local --enumerate --api-token <TOKEN>
   ```
   This confirms versions, discovers additional users (e.g., via Author ID brute-forcing), checks for known CVEs, and identifies misconfigurations like XML-RPC being enabled.

## WordPress User Roles

| Role | Capabilities |
|------|-------------|
| Administrator | Full access — add/delete users/posts, edit source code |
| Editor | Publish and manage all posts |
| Author | Publish and manage own posts |
| Contributor | Write/manage own posts but cannot publish |
| Subscriber | Browse posts and edit own profile |

Admin = code execution. Editors/Authors may access vulnerable plugins normal users can't.

## Key Findings from the Example Environment

- **WordPress 5.8** — 3 known vulns (Data Exposure via REST API, Authenticated XSS, Lodash update)
- **Theme**: Transport Gravity (child of Business Gravity)
- **Plugins found**:
  - **mail-masta 1.0** — Unauthenticated LFI + SQL Injection
  - **wpDiscuz 7.0.4** — Unauthenticated RCE
  - **Contact Form 7 5.4.2**
  - **WP Sitemap Page** — found by manually browsing (not caught by automated scan)
- **Users**: `admin`, `john` (confirmed valid)
- **Misconfigs**: Directory listing enabled, XML-RPC enabled (brute-force vector), upload directory exposed

## WPScan Key Flags

| Flag | Purpose |
|------|---------|
| `--url` | Target URL (required) |
| `--enumerate` | Default enum (vuln plugins, themes, users, media, backups) |
| `--enumerate ap` | Enumerate ALL plugins (not just vulnerable ones) |
| `--enumerate at` | Enumerate ALL themes |
| `--enumerate u` | Enumerate users |
| `--api-token` | WPVulnDB API token for CVE data (free: 25 req/day) |
| `-t` | Number of threads (default: 5) |
| `--detection-mode` | `mixed` (default), `passive`, or `aggressive` |

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Find flag.txt in an accessible directory | flag in `wp-content/uploads/2021/08/flag.txt` | WPScan reports directory listing on `/wp-content/uploads/` → browse subdirs `2021/08/` |
| Q2: Another installed plugin (3 words) | WP Sitemap Page | Browse to root page → click "Shipping Industry News" → plugin visible in page source |
| Q3: Version number of that plugin | Check `readme.txt` at `/wp-content/plugins/wp-sitemap-page/readme.txt` → `Stable tag` value |

## Key Takeaways

- **Manual + automated enumeration together** — WPScan missed wpDiscuz and Contact Form 7 in the example; manual browsing of multiple pages caught them
- **Browse multiple pages** when grepping for plugins — some only load on specific page types (comment plugins on posts, sitemap plugins on sitemap pages)
- **readme.txt in plugin directories** is the fastest way to fingerprint plugin versions (`Stable tag` field)
- **WordPress login page leaks valid usernames** by design — different error messages for valid vs invalid usernames
- **Don't jump to exploitation during enumeration** — finish the full discovery phase first; you might miss critical vulns by tunneling on the first one you find
- **54% of WordPress vulns come from plugins**, 31.5% from core, 14.5% from themes — plugins are your primary attack surface
- **XML-RPC enabled** = brute-force vector via WPScan or Metasploit (`wordpress_xmlrpc_login`)

## Gotchas

- WPScan's default `--enumerate` only checks *vulnerable* plugins passively — use `--enumerate ap` to catch all plugins including those without known CVEs
- Child themes (like Transport Gravity) can be misidentified as their parent (Business Gravity) during manual enum — WPScan clarifies
- The WPVulnDB free API token is limited to 25 requests/day — don't waste them on repeated scans
- Directory listing on `/wp-content/uploads/` is worth browsing deeply through year/month subdirs — flags, backups, and sensitive files can be buried several levels down
