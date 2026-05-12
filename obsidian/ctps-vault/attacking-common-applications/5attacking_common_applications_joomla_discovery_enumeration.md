# Section 5 — Joomla - Discovery & Enumeration

## ID
533

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 5 — Joomla - Discovery & Enumeration

## Description
Covers Joomla fingerprinting, version enumeration via README.txt and manifests, scanning with droopescan and JoomlaScan, and brute-forcing the admin login.

## Tags
joomla, cms, enumeration, droopescan, brute-force, fingerprinting

## Commands
- `curl -s http://<TARGET>/ | grep Joomla`
- `curl -s http://<TARGET>/README.txt | head -n 5`
- `curl -s http://<TARGET>/administrator/manifests/files/joomla.xml | xmllint --format -`
- `droopescan scan joomla --url http://<TARGET>/`
- `python2.7 joomlascan.py -u http://<TARGET>`
- `python3 joomla-brute.py -u http://<TARGET> -w <WORDLIST> -usr admin`
- `sudo pip3 install droopescan`

## What This Section Covers
How to discover, fingerprint, and enumerate Joomla installations. Joomla powers ~3% of all websites and has ~2.7 million installs. The section covers version identification through multiple files, automated scanning with droopescan and JoomlaScan, and brute-forcing the admin login at `/administrator/`. Unlike WordPress, Joomla doesn't leak valid usernames through login error messages.

## Methodology

1. **Confirm Joomla is in use** — grep page source for "Joomla" in the generator meta tag:
   ```
   curl -s http://dev.inlanefreight.local/ | grep Joomla
   ```
   Also check `/robots.txt` — Joomla's robots.txt disallows `/administrator/`, `/bin/`, `/cache/`, `/cli/`, `/components/`, `/includes/`, etc.

2. **Fingerprint the version** — multiple methods, try in order:

   **README.txt** (most reliable):
   ```
   curl -s http://dev.inlanefreight.local/README.txt | head -n 5
   ```

   **joomla.xml manifest** (exact version):
   ```
   curl -s http://dev.inlanefreight.local/administrator/manifests/files/joomla.xml | xmllint --format -
   ```

   **cache.xml** (approximate version):
   ```
   curl -s http://dev.inlanefreight.local/plugins/system/cache/cache.xml
   ```

   Also check JavaScript files in `media/system/js/` directory.

3. **Scan with droopescan** for version narrowing and interesting URLs:
   ```
   droopescan scan joomla --url http://dev.inlanefreight.local/
   ```

4. **Scan with JoomlaScan** (Python 2.7) for component enumeration and directory discovery:
   ```
   python2.7 joomlascan.py -u http://dev.inlanefreight.local
   ```
   Finds installed components (`com_*`), explorable directories, and license/XML files.

5. **Brute-force admin login** at `/administrator/index.php` — default admin account is `admin` (password set at install):
   ```
   python3 joomla-brute.py -u http://dev.inlanefreight.local -w /usr/share/metasploit-framework/data/wordlists/http_default_pass.txt -usr admin
   ```

## Joomla vs WordPress — Key Differences

| Feature | Joomla | WordPress |
|---------|--------|-----------|
| Admin panel | `/administrator/` | `/wp-admin/` |
| User enumeration via login | No (generic error) | Yes (different messages) |
| Version in page source | generator meta tag | generator meta tag |
| Plugin directory | `/components/`, `/plugins/` | `/wp-content/plugins/` |
| Default admin user | `admin` | `admin` |
| Market share | ~3% | ~32.5% |

## Joomla Version Fingerprinting Locations

| File/Path | Info Provided |
|-----------|--------------|
| `/README.txt` | Major version + version history link |
| `/administrator/manifests/files/joomla.xml` | Exact version in `<version>` tag |
| `/plugins/system/cache/cache.xml` | Approximate version |
| `media/system/js/` | JS files may reveal version |
| Page source `<meta name="generator">` | Confirms Joomla, sometimes version |

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Joomla version on app.inlanefreight.local | 3.10.0 | `curl -s app.inlanefreight.local/README.txt \| head -n 4` → "Joomla! 3.10 version history" → format as 3.10.0 |
| Q2: Password for admin user | turnkey | `python3 joomla-brute.py -u http://app.inlanefreight.local -w /usr/share/metasploit-framework/data/wordlists/http_default_pass.txt -usr admin` |

## Key Takeaways

- **Joomla login page doesn't enumerate users** — gives a generic "Username and password do not match" for both invalid users and wrong passwords, unlike WordPress
- **README.txt is the quickest version check** — but `joomla.xml` gives the most precise version number
- **droopescan has limited Joomla support** — it narrows version ranges but doesn't enumerate plugins/themes as thoroughly as WPScan does for WordPress
- **Default admin account is always `admin`** — password is set at install, so weak/default passwords are your main brute-force target
- **Try small wordlists first** — `http_default_pass.txt` from Metasploit is a fast first check before jumping to rockyou.txt
- **Joomla components follow the `com_*` naming convention** — enumerate via `index.php?option=com_<name>` or browse `/components/` and `/administrator/components/`

## Gotchas

- JoomlaScan requires Python 2.7 — use `pyenv shell 2.7` if you have pyenv set up, and install deps with `python2.7 -m pip install urllib3 certifi bs4`
- The README.txt may say "version 3.x" generically — cross-reference with `joomla.xml` for the exact version (e.g., 3.9.4 vs 3.10.0)
- droopescan may return a range of possible versions (e.g., 3.8.7 through 3.8.13) — use `joomla.xml` to confirm the exact one
- The joomla-brute.py script needs to be cloned from GitHub (`https://github.com/ajnik/joomla-bruteforce.git`) — it's not pre-installed
