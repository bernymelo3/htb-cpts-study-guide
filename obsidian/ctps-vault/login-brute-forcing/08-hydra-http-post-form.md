# NOTE — Login Forms (HTTP POST Form Brute Forcing)

## ID
515

## Module
Password Attacks

## Kind
notes

## Title
Section 8 — Login Forms (http-post-form with Hydra)

## Description
Use Hydra’s `http-post-form` module to brute‑force a web login form by inspecting HTML, capturing POST parameters, and defining failure/success conditions.

## Tags
hydra, http-post-form, web-forms, brute-forcing, login-form

## Commands
- `curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/master/Usernames/top-usernames-shortlist.txt`
- `curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/2023-200_most_used_passwords.txt`
- `hydra -L top-usernames-shortlist.txt -P 2023-200_most_used_passwords.txt -f <TARGET_IP> -s <PORT> http-post-form "/:username=^USER^&password=^PASS^:F=Invalid credentials"`

## What This Section Covers
Most web applications use custom login forms instead of Basic Auth. This section teaches how to brute‑force such forms with Hydra’s `http-post-form` module. You learn to inspect the form’s HTML (method, action, field names), capture a POST request using browser developer tools, and construct the Hydra parameters string with failure conditions (`F=...`) or success conditions (`S=...`). A live lab demonstrates cracking a login form that returns “Invalid credentials” on failure.

## Methodology
1. **Spawn the target instance** — get IP and port.
2. **Inspect the login form** — open the URL, right‑click → Inspect. Note:
   - Form `method` (usually POST)
   - Form `action` (target path, e.g., `/`)
   - Input `name` attributes (e.g., `username`, `password`)
3. **Test a dummy login** — open Developer Tools (F12) → Network tab, submit any credentials. Find the POST request, verify parameters and the error message returned (e.g., “Invalid credentials”).
4. **Download wordlists** — usernames (`top-usernames-shortlist.txt`) and passwords (`2023-200_most_used_passwords.txt`).
5. **Construct Hydra command**:
   - `-L` for username list
   - `-P` for password list
   - `-f` to stop on first success
   - `http-post-form` with `"path:params:condition"`
   - Condition example: `F=Invalid credentials` (failure string)
6. **Run Hydra** — it will test combinations and output valid credentials when found.
7. **Log in via browser** — use the discovered credentials to retrieve the flag.

## Multi-step Workflow (example with live target)
```bash
# Download wordlists
curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/master/Usernames/top-usernames-shortlist.txt
curl -s -O https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/2023-200_most_used_passwords.txt

# Run Hydra against the target
hydra -L top-usernames-shortlist.txt -P 2023-200_most_used_passwords.txt -f 154.57.164.75 -s 31982 http-post-form "/:username=^USER^&password=^PASS^:F=Invalid credentials"

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|login-brute-forcing]]  
← [[07-basic-http-auth]] | [[09-medusa]] →
<!-- AUTO-LINKS-END -->
