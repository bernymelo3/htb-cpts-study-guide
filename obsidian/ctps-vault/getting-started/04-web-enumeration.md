---
id: gs-104
module: Getting Started
kind: note
title: "Web Enumeration & Discovery"
description: "Directory/file brute-force, subdomain enumeration, HTTP headers, technology fingerprinting, robots.txt, source code analysis, SSL certificates."
tags: [web-enumeration, gobuster, ffuf, whatweb, curl, robots-txt, source-code, ssl-certificate]
---

# Web Enumeration & Discovery

## Directory/File Brute-Force (Gobuster)

### Basic Directory Brute-Force
```bash
gobuster dir -u http://$TARGET_IP/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 40 -q
```
**Flags:**
- `-u` = target URL
- `-w` = wordlist
- `-t 40` = 40 concurrent threads (speed; adjust for stability)
- `-q` = quiet (only show results, not "Not Found")

### Recursive Brute-Force (Find Deeper Paths)
```bash
gobuster dir -u http://$TARGET_IP/ -w /usr/share/seclists/Discovery/Web-Content/big.txt -t 40 -r
# -r = follow redirects
```

### Include Status Codes
```bash
gobuster dir -u http://$TARGET_IP/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -s 200,301,302,307,401,403
# -s = only show these HTTP status codes
```

### Check for .php Extensions
```bash
gobuster dir -u http://$TARGET_IP/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -x php,html,txt,bak
# -x = append extensions to wordlist entries
```

**Common wordlists:**
- `/usr/share/seclists/Discovery/Web-Content/common.txt` (fast, ~5K entries)
- `/usr/share/seclists/Discovery/Web-Content/big.txt` (~20K entries)
- `/usr/share/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt` (~220K entries, slow)

---

## Subdomain Enumeration (DNS)

### Gobuster DNS Mode
```bash
gobuster dns -d example.com -w /usr/share/seclists/Discovery/DNS/namelist.txt -t 40
```

### Add DNS Server (if default fails)
```bash
echo "nameserver 1.1.1.1" | sudo tee /etc/resolv.conf
gobuster dns -d example.com -w /usr/share/seclists/Discovery/DNS/namelist.txt
```

**Look for:**
- `admin.example.com`
- `api.example.com`
- `blog.example.com`
- `vpn.example.com`
- `internal.example.com`

---

## Technology Fingerprinting

### Whatweb (Quick ID)
```bash
whatweb $TARGET_IP
# Outputs: CMS version, server, JS libraries, email addresses, etc.
```

### HTTP Headers (Manual)
```bash
curl -I http://$TARGET_IP
# Look for: Server, X-Powered-By, X-AspNet-Version, X-Frame-Options
```

**Example:**
```
Server: Apache/2.4.41 (Ubuntu)
X-Powered-By: PHP/7.4.3
X-Frame-Options: SAMEORIGIN
```

**What this tells you:**
- Apache 2.4.41 on Ubuntu → searchsploit for known vulns
- PHP 7.4.3 → check for file inclusion, RCE exploits
- X-Frame-Options present → app has some security awareness

### Banner Grabbing (Non-HTTP)
```bash
nc -nv $TARGET_IP 8080
# Wait for service banner
```

---

## Analyze Source Code & Comments

### View Page Source (Browser)
Press `Ctrl+U` or view-source; look for:

```html
<!-- TODO: Remove this admin panel later -->
<!-- username: admin, password: password123 -->
<script src="/admin.php?key=secret"></script>
<base href="/admin-panel-hidden/">
```

### Download & Analyze
```bash
curl http://$TARGET_IP/ > index.html
grep -i "password\|key\|secret\|admin\|TODO" index.html
```

---

## Robots.txt & Sitemap

### Check robots.txt
```bash
curl http://$TARGET_IP/robots.txt
```

**Example:**
```
User-agent: *
Disallow: /admin/
Disallow: /uploaded_files/
Disallow: /private/
```

→ Check these disallowed paths; they're often accessible anyway.

### Check sitemap.xml
```bash
curl http://$TARGET_IP/sitemap.xml
```

---

## SSL/TLS Certificates

### View in Browser
Click lock icon → Certificate Details

### Extract via OpenSSL
```bash
echo | openssl s_client -connect $TARGET_IP:443 2>/dev/null | grep -A 20 "Subject:"
```

**Look for:**
- Email addresses (phishing/social eng)
- Organization name (company domain)
- Subject Alt Names (subdomains)
- Certificate validity (expired = misconfiguration)

---

## Common Findings & What They Mean

| Finding | Implication | Next Step |
|---------|-------------|-----------|
| `/admin` directory found | Admin panel likely exists | Try default creds, look for login bypass |
| `/config.php` discoverable | Config file accessible (dangerous) | Download and check for DB creds, API keys |
| `/backup`, `/old`, `/test` dirs | Abandoned or test code | May contain hardcoded passwords, outdated vulns |
| WordPress detected | CMS present with known attack surface | Check for plugins, vulnerable versions, wpsccan enum |
| PHP version disclosed | Direct CVE search | e.g., PHP 7.0.33 → RCE in file uploads, type confusion |
| Apache version + status | `mod_status` may be enabled | Visit `/server-status` for running processes, source IPs |
| Robots.txt exists with entries | Hint at sensitive paths | Check all Disallowed entries; they're often accessible |

---

## Practical Web Enum Workflow

```bash
# 1. Identify web server + tech
whatweb $TARGET_IP
curl -I http://$TARGET_IP | grep Server

# 2. Check robots.txt + sitemap
curl http://$TARGET_IP/robots.txt
curl http://$TARGET_IP/sitemap.xml

# 3. Brute-force directories
gobuster dir -u http://$TARGET_IP/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 40 -q

# 4. View source for comments / scripts
curl http://$TARGET_IP/ | grep -i "script\|<!--\|admin\|password"

# 5. For each discovered path, repeat gobuster + source analysis
gobuster dir -u http://$TARGET_IP/admin/ -w /usr/share/seclists/Discovery/Web-Content/common.txt -q

# 6. Try default credentials on login pages
# admin:admin, admin:password, guest:guest, etc.

# 7. Search for CMS/app version exploits
# e.g., if WordPress detected: searchsploit WordPress 5.1
```

---

## Enum Speed Tips (Exam)

1. **Start gobuster with big.txt in background** while analyzing first 10 results manually
2. **Filter HTTP 200s only** to reduce noise: `-s 200` (unless looking for redirects)
3. **Use EyeWitness** if you find many URLs and need screenshots fast:
   ```bash
   eyewitness -f urls.txt --web
   ```
4. **Prioritize `/admin`, `/login`, `/config`, `/backup`, `/upload`** — high-value paths
5. **Check source on every page** — comments often reveal design intent + sometimes credentials

---

## Related Notes

[[../web-recon/]] — Deeper passive recon (DNS, WHOIS, ASN lookup)  
[[../web-attacks/]] — Exploitation of web vulns (SQLi, XSS, CSRF)  
[[05-public-exploits]] — Finding & running exploits for discovered tech
