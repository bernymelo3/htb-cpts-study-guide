# NOTE — Module Summary: Login Brute Forcing

## ID
520

## Module
Password Attacks

## Kind
notes

## Title
Module Summary — Login Brute Forcing (Full Reference)

## Description
Complete reference covering brute‑force attack types, Hydra and Medusa syntax, custom wordlist generation (username-anarchy, CUPP), and a full skills assessment walkthrough chaining HTTP Basic Auth → SSH → FTP.

## Tags
summary, reference, hydra, medusa, wordlists, skills-assessment

## Commands
- `hydra -L usernames.txt -P passwords.txt <IP> -s <PORT> http-get /`
- `hydra -l admin -P passwords.txt <IP> http-post-form "/login:username=^USER^&password=^PASS^:F=Invalid credentials"`
- `hydra -l sshuser -P passwords.txt <IP> -s <PORT> ssh`
- `hydra -L thomas_usernames.txt -P passwords.txt localhost ftp`
- `medusa -h <IP> -u admin -P passwords.txt -M ssh -n <PORT>`
- `./username-anarchy FirstName LastName > usernames.txt`
- `cupp -i`
- `netstat -tlnp` / `ss -tlnp`
- `ssh user@<IP> -p <PORT>`
- `ftp ftp://user:pass@localhost`

## What This Section Covers
This is a consolidated reference for the entire Login Brute Forcing module. It includes: types of brute‑force attacks (dictionary, credential stuffing, password spraying, hybrid), detailed Hydra usage (Basic Auth, POST forms, SSH, FTP), Medusa syntax, custom wordlist generation with username-anarchy and CUPP, common SecLists locations, and a step‑by‑step skills assessment that chains HTTP Basic Auth → SSH → FTP to retrieve a final flag. The assessment demonstrates pivoting, internal service enumeration, and username permutation generation.

## Methodology (Skills Assessment Recap)
1. **Identify service** — use `curl` or `nmap -sV`; non‑standard port may not be HTTP.
2. **Brute force HTTP Basic Auth** — Hydra with `http-get /` to find credentials.
3. **Login via browser** — retrieve a username for the next stage.
4. **Identify SSH on non‑standard port** — `nmap -sV` reveals SSH.
5. **Brute force SSH** — Hydra with known username and password list.
6. **SSH into target** — enumerate files (`IncidentReport.txt`, `passwords.txt`, `username-anarchy/`).
7. **Generate username permutations** — `username-anarchy` from the suspect’s name.
8. **Locate FTP service** — `netstat -tlnp` shows port 21 listening locally (filtered externally).
9. **Brute force FTP from inside** — Hydra against `localhost` with generated usernames and discovered password list.
10. **Retrieve flag** — `ftp localhost`, `get flag.txt`, `cat flag.txt`.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Final flag from Skills Assessment | `HTB{brut3f0rc1ng_succ3ssful}` | `cat flag.txt` after FTP download |

## Key Takeaways
- **Always verify the service** before brute forcing — a port number does not guarantee the protocol. Use `nmap -sV` or `curl -i`.
- **Internal‑only services** are common — if a port shows as filtered externally, SSH in and check locally.
- **username‑anarchy** is essential for generating login name permutations from a real name.
- **Chain services** — each compromised system provides credentials or intel for the next (web → SSH → FTP).
- **Wordlist selection matters** — for CTF exercises, use the provided lists; for real assessments, tailor wordlists to the target.
- **Hydra vs. Medusa** — Hydra is more common, but Medusa is useful as a fallback on compromised hosts.
- **Password spraying** avoids lockouts by trying one password across many usernames — not covered in depth here but important.

## Gotchas
- **Non‑HTTP services on port 80/443** — `curl` may return `Received HTTP/0.9 when not allowed`, indicating a non‑HTTP service. Use `nmap -sV`.
- **FTP on localhost only** — external `nmap` shows filtered; you must attack from inside the SSH session.
- **Wordlist paths** — on Pwnbox, SecLists are under `/opt/useful/seclists/`. Use those paths instead of downloading fresh copies.
- **username‑anarchy** may not be pre‑installed; the skills assessment provides it in the home directory after SSH access.
- **Hydra’s `http-post-form`** requires exact parameter names and failure strings — always capture a failed login response first.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|login-brute-forcing]]  
← [[10-ssh-ftp-brute]]
<!-- AUTO-LINKS-END -->
