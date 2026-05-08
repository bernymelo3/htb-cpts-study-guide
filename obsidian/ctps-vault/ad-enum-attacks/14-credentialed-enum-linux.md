## ID
905

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 14 — Credentialed Enumeration - from Linux

## Description
Covers post-foothold AD enumeration from a Linux attack host using CrackMapExec, SMBMap, rpcclient, Impacket (psexec.py/wmiexec.py), Windapsearch, and BloodHound.py — all requiring at minimum a valid domain user credential.

## Tags
active-directory, crackmapexec, smbmap, rpcclient, impacket, bloodhound, ldap

## Commands
- sudo crackmapexec smb <DC_IP> -u <USER> -p <PASS> --users
- sudo crackmapexec smb <DC_IP> -u <USER> -p <PASS> --groups
- sudo crackmapexec smb <TARGET_IP> -u <USER> -p <PASS> --loggedon-users
- sudo crackmapexec smb <DC_IP> -u <USER> -p <PASS> --shares
- sudo crackmapexec smb <DC_IP> -u <USER> -p <PASS> -M spider_plus --share '<SHARE_NAME>'
- smbmap -u <USER> -p <PASS> -d <DOMAIN> -H <DC_IP>
- smbmap -u <USER> -p <PASS> -d <DOMAIN> -H <DC_IP> -R '<SHARE>' --dir-only
- rpcclient -U "<USER>%<PASS>" <DC_IP> -c "enumdomusers"
- rpcclient -U "<USER>%<PASS>" <DC_IP> -c "queryuser <HEX_RID>"
- psexec.py <DOMAIN>/<USER>:'<PASS>'@<TARGET_IP>
- wmiexec.py <DOMAIN>/<USER>:'<PASS>'@<TARGET_IP>
- python3 windapsearch.py --dc-ip <DC_IP> -u <USER>@<DOMAIN> -p <PASS> --da
- python3 windapsearch.py --dc-ip <DC_IP> -u <USER>@<DOMAIN> -p <PASS> -PU
- sudo bloodhound-python -u '<USER>' -p '<PASS>' -ns <DC_IP> -d <DOMAIN> -c all

## What This Section Covers
Once you have a valid domain user credential (cleartext password, NTLM hash, or SYSTEM on a domain-joined host), you can perform deep AD enumeration from Linux. This section walks through six toolsets — CrackMapExec for user/group/share/session enumeration, SMBMap for share access and recursive directory listing, rpcclient for RID-based user lookups, Impacket's psexec.py and wmiexec.py for remote execution, Windapsearch for LDAP-based privileged user discovery, and BloodHound.py for full attack path ingestion.

## Methodology
1. **CME — Enumerate domain users:** Run `sudo crackmapexec smb <DC_IP> -u <USER> -p <PASS> --users` to list all domain users with their `badpwdcount` (useful for filtering spray targets away from lockout).
2. **CME — Enumerate domain groups:** Run `--groups` to list all groups with member counts. Note high-value groups: Domain Admins, Backup Operators, Executives, any custom IT/admin groups.
3. **CME — Check logged-on users:** Target file servers or jump hosts with `--loggedon-users`. Look for `(Pwn3d!)` which means your user is local admin on that host. Identify privileged users like `svc_qualys` or `lab_adm` whose sessions you could hijack.
4. **CME — Enumerate shares:** Run `--shares` against the DC and other hosts. Look for non-default shares with READ/WRITE access (e.g., `Department Shares`, `User Shares`, `ZZZ_archive`).
5. **CME — Spider shares for files:** Use `-M spider_plus --share '<SHARE>'` to crawl readable shares and output a JSON file at `/tmp/cme_spider_plus/<IP>.json` listing all accessible files — look for scripts, configs, or files containing hardcoded creds.
6. **SMBMap — Verify share permissions:** Run `smbmap -u <USER> -p <PASS> -d <DOMAIN> -H <DC_IP>` for a clean permissions view (READ ONLY / NO ACCESS). Use `-R '<SHARE>' --dir-only` for recursive directory listing.
7. **rpcclient — RID-based user enumeration:** Connect with `rpcclient -U "<USER>%<PASS>" <DC_IP>`, then run `enumdomusers` to get all users with RIDs. Use `queryuser <HEX_RID>` to pull detailed info on specific users (logon count, password last set, etc.).
8. **Impacket — psexec.py:** If you have local admin creds on a target, use `psexec.py <DOMAIN>/<USER>:'<PASS>'@<IP>` to get a SYSTEM shell. It uploads to ADMIN$, creates a service via RPC, communicates over a named pipe.
9. **Impacket — wmiexec.py:** Stealthier than psexec — uses WMI for semi-interactive shell execution. Runs as the user you authenticate with (not SYSTEM), drops no files, generates fewer logs. Each command spawns a new `cmd.exe` (event ID 4688).
10. **Windapsearch — Domain Admins and privileged users:** Use `--da` flag to list Domain Admins, or `-PU` for recursive nested group membership across all privileged groups (Domain Admins, Enterprise Admins, etc.).
11. **BloodHound.py — Full domain ingestion:** Run `sudo bloodhound-python -u '<USER>' -p '<PASS>' -ns <DC_IP> -d <DOMAIN> -c all` to collect users, groups, computers, sessions, ACLs, GPOs, trusts. Upload the resulting JSON files (or zip them) into the BloodHound GUI for graph-based attack path analysis.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: What AD User has a RID equal to Decimal 1170? | **mmorgan** | SSH to attack host → `rpcclient -U "" -N 172.16.5.5` → convert 1170 decimal to hex = `0x492` → `queryuser 0x492` returns `mmorgan` (Matthew Morgan) |
| Q2: What is the membercount: of the "Interns" group? | **10** | `sudo crackmapexec smb 172.16.5.5 -u forend -p Klmcargo2 --groups` → grep output for `Interns` → `membercount: 10` |

## Key Takeaways
- All tools in this section require at minimum a valid domain user credential — even low-privilege access unlocks massive enumeration capability.
- `(Pwn3d!)` in CME output means your user is local admin on that host — immediate lateral movement opportunity via psexec/wmiexec.
- RIDs are hex in rpcclient output but questions may ask in decimal — convert with `printf '%x\n' <DECIMAL>` or Python `hex()` before querying.
- `spider_plus` is underrated — it silently crawls every readable file in a share and dumps to JSON, much faster than manual browsing.
- wmiexec.py is stealthier than psexec.py (no file drops, runs as user not SYSTEM, fewer logs) but each command spawns a new `cmd.exe` process which creates event 4688 entries.
- BloodHound `-PU` (privileged users) with Windapsearch catches nested group membership that manual enumeration would miss — great for finding hidden admin paths.
- Always save tool output to files (`| tee <filename>`) — you'll need it for reporting and cross-referencing later.

## Gotchas
- rpcclient NULL sessions (`-U "" -N`) may or may not work depending on the domain config — if they fail, authenticate with valid creds instead (`-U "user%pass"`).
- CME requires `sudo` for many operations — forgetting it produces silent failures or permission errors.
- BloodHound.py outputs JSON files to the current working directory — run it from a dedicated folder or you'll scatter files everywhere.
- psexec.py drops a randomly-named executable to ADMIN$ and creates a service — this is noisy and will be caught by most EDR. Use wmiexec.py for stealth.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[13-enumerating-security-controls]] | [[15-credentialed-enum-windows]] →
<!-- AUTO-LINKS-END -->
