# Section 17 — GitLab - Discovery & Enumeration

## ID
908

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 17 — GitLab - Discovery & Enumeration

## Description
Covers discovering and fingerprinting GitLab instances, registering accounts to access internal repositories, enumerating valid usernames via the registration form, and mining repos for hardcoded secrets like database passwords and SSH keys.

## Tags
gitlab, enumeration, git, credential-harvesting, user-enumeration, devops

## Commands
- `echo '<TARGET_IP> gitlab.inlanefreight.local' | sudo tee -a /etc/hosts`
- `curl -s http://gitlab.inlanefreight.local/explore`
- `curl -s http://gitlab.inlanefreight.local/help`

## What This Section Covers
GitLab is an open-source web-based Git repository manager (similar to GitHub and Bitbucket) used by companies like Goldman Sachs, HackerOne, Nvidia, and Siemens. From a pentester's perspective, GitLab instances are treasure troves — repos may contain hardcoded credentials, SSH keys, API tokens, database passwords, and infrastructure details in configuration files or commit history. The section covers how to discover GitLab, determine its version, register an account for deeper access, enumerate valid usernames, and systematically hunt for secrets across public and internal repositories.

## Background — GitLab at a Glance

- Originally written in Ruby; current stack is Go, Ruby on Rails, and Vue.js
- First launched in 2014; 1,466 employees, 30M+ registered users across 66 countries, $150M revenue (2020)
- Free/open-source with a paid enterprise edition
- Repositories can be **public** (no auth), **internal** (authenticated users only), or **private** (specific users only)
- 2FA is **disabled by default**
- Notable CVEs against versions 12.9.0, 11.4.7, 13.10.3, 13.9.3, and 13.10.2

## Methodology

### Phase 1 — Discovery & Fingerprinting

1. **Add vHost to `/etc/hosts`:**
   ```
   echo '<TARGET_IP> gitlab.inlanefreight.local' | sudo tee -a /etc/hosts
   ```

2. **Browse to the GitLab URL** — the login page displays the GitLab logo, immediately confirming the application.

3. **Version fingerprinting** — the only reliable way is browsing to `/help` **while logged in**. The version appears at the top of the help page. If you can't log in, options are limited: try a low-risk exploit for version detection, look for dates on the page, or check first public commits. **Do not blindly throw exploits at it** without version confirmation.

### Phase 2 — Unauthenticated Enumeration

4. **Browse `/explore`** — check for public projects visible without authentication. Look for:
   - Hardcoded credentials in config files
   - SSH private keys
   - API keys/tokens
   - Infrastructure details (hostnames, IPs, network diagrams)
   - Production source code (review for vulnerabilities)

5. **Enumerate usernames via the registration form** — browse to `/users/sign_up` and try common usernames (root, admin, etc.). GitLab returns specific errors:
   - `"Username is already taken"` → user exists
   - `"Email has already been taken"` → email is registered
   - This works **even if sign-up is disabled** — you can still browse to `/users/sign_up` and enumerate, you just can't complete registration.

### Phase 3 — Register & Authenticate

6. **Register an account** — if sign-up is enabled and doesn't require a company email or admin approval, register with any email and log in.

7. **Re-check `/explore`** — after logging in, **internal repositories** become visible that weren't shown before. In this lab, the `Inlanefreight website` project appears only after authentication.

8. **Browse to `/help`** — now that you're logged in, the exact version number is shown at the top of the page.

### Phase 4 — Secret Hunting

9. **Dig through every accessible repo** — check:
   - Configuration files (`.xml`, `.yml`, `.env`, `.conf`, `.ini`, `.json`)
   - Commit history (secrets may have been committed then "removed" — they're still in Git history)
   - Scripts and automation files
   - README files and wiki pages
   - Snippets (accessible via the Snippets tab)

10. **Use GitLab's search function** — search for keywords like `password`, `secret`, `key`, `token`, `api`, `credential`, `jdbc`, `connection_string` across all accessible projects.

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: GitLab version number | 13.10.2 | Register account → log in → browse to `/help` → version at top of page |
| Q2: PostgreSQL database password | postgres | Log in → Projects → Explore projects → open "Inlanefreight dev" → open `phpunit_pgsql.xml` → password is in the XML config |

## Key Takeaways

- **GitLab's value is in the data** — even without a code exploit, repos routinely leak credentials, keys, and infrastructure details
- **Public repos are visible without auth, but internal repos require login** — always try to register an account for deeper access
- **Username enumeration works even with sign-up disabled** — the `/users/sign_up` page still responds with `"Username is already taken"` errors
- **2FA is off by default** — brute-force and credential stuffing attacks are viable if you have a username list
- **Commit history retains deleted secrets** — even if a password was removed in a later commit, it's still in the Git log; tools like `truffleHog` and `GitLeaks` automate this search
- **Version confirmation requires authentication** — `/help` only shows the version when logged in, which is important for deciding which exploits to attempt

## Gotchas

- The lab uses port 8081 for GitLab in some walkthrough examples but the default port in the section reading — always confirm the actual port via Nmap
- Don't blindly fire exploits without version confirmation — stick to secret hunting if you can't determine the version
- Some GitLab instances restrict registration to company email domains or require admin approval — check the error messages carefully
- The `phpunit_pgsql.xml` file with the database password is in the **public** "Inlanefreight dev" project, not the internal one — even unauthenticated access could find it via `/explore`
