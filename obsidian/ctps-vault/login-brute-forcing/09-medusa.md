# NOTE — Medusa

## ID
516

## Module
Password Attacks

## Kind
notes

## Title
Section 9 — Medusa (Parallel Login Brute‑Forcer)

## Description
Introduction to Medusa, a fast, massively parallel login brute‑forcer supporting many protocols (SSH, FTP, HTTP, RDP, MySQL, etc.), covering installation, command syntax, modules, and practical examples.

## Tags
medusa, brute-forcing, password-attacks, tool, parallel-attacks

## Commands
- `medusa -h`
- `sudo apt-get install medusa`
- `medusa -h 192.168.0.100 -U usernames.txt -P passwords.txt -M ssh`
- `medusa -H web_servers.txt -U usernames.txt -P passwords.txt -M http -m GET`
- `medusa -h 10.0.0.5 -U usernames.txt -e ns -M service_name`
- `medusa -M http -h www.example.com -U users.txt -P passwords.txt -m DIR:/login.php -m FORM:username=^USER^&password=^PASS^`

## What This Section Covers
Medusa is a parallel login brute‑forcer similar to Hydra but with a focus on massive parallelism and modular design. This section explains Medusa’s installation, core command syntax (target, credential, module options), and the most useful modules (SSH, FTP, HTTP, RDP, MySQL, etc.). It provides concrete examples: attacking a single SSH server, testing multiple web servers with Basic Auth, and checking for empty or default passwords (`-e ns`). Unlike Hydra, Medusa uses `-M` to specify the module and can test multiple hosts from a file (`-H`).

## Methodology
1. **Install Medusa** — `sudo apt-get install medusa` if not already present.
2. **Choose a target** — single host (`-h`) or list of hosts (`-H`).
3. **Provide credentials** — single username (`-u`) or list (`-U`); single password (`-p`) or list (`-P`).
4. **Select a module** — `-M` with module name (ssh, ftp, http, rdp, mysql, etc.).
5. **Set parallelism** — `-t` for number of threads (default varies).
6. **Add module options** — `-m` for module‑specific parameters (e.g., HTTP form details).
7. **Enable fast stop** — `-f` (stop on first success for current host) or `-F` (stop on any host).
8. **Run and analyze output** — valid credentials are printed when found.

## Multi-step Workflow (examples)

### Example 1: SSH Brute Force
```bash
medusa -h 192.168.0.100 -U usernames.txt -P passwords.txt -M ssh

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|login-brute-forcing]]  
← [[08-hydra-http-post-form]] | [[10-ssh-ftp-brute]] →
<!-- AUTO-LINKS-END -->
