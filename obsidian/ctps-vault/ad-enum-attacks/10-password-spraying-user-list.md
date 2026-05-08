## ID
901

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 10 — Password Spraying - Making a Target User List

## Description
Covers multiple methods to build a valid domain user list for password spraying — SMB NULL sessions, LDAP anonymous binds, Kerbrute username enumeration, and credentialed enumeration with CrackMapExec.

## Tags
active-directory, user-enumeration, kerbrute, password-spraying, smb-null-session, ldap-anonymous-bind

## Commands
- enum4linux -U <DC_IP> | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"
- rpcclient -U "" -N <DC_IP> -c enumdomusers
- crackmapexec smb <DC_IP> --users
- ldapsearch -h <DC_IP> -x -b "DC=<DOMAIN>,DC=<TLD>" -s sub "(&(objectclass=user))" | grep sAMAccountName: | cut -f2 -d" "
- ./windapsearch.py --dc-ip <DC_IP> -u "" -U
- kerbrute userenum -d <DOMAIN> --dc <DC_IP> <WORDLIST>
- crackmapexec smb <DC_IP> -u <USER> -p <PASS> --users

## What This Section Covers
Before password spraying, you need a list of valid domain usernames. This section walks through building that list using unauthenticated methods (SMB NULL sessions, LDAP anonymous binds, Kerbrute) and credentialed enumeration (CrackMapExec with valid creds). Kerbrute is highlighted as the stealthiest option since it uses Kerberos Pre-Authentication and does not generate Windows event ID 4625 (logon failure).

## Methodology
1. **SMB NULL Session — enum4linux:** Run `enum4linux -U <DC_IP>` and pipe through `grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"` to extract clean usernames.
2. **SMB NULL Session — rpcclient:** Connect with `rpcclient -U "" -N <DC_IP>`, then run `enumdomusers` to list all domain users with their RIDs.
3. **SMB NULL Session — CrackMapExec:** Run `crackmapexec smb <DC_IP> --users` to get usernames plus `badpwdcount` and `baddpwdtime` — useful for filtering out accounts near lockout.
4. **LDAP Anonymous Bind — ldapsearch:** Use `ldapsearch -h <DC_IP> -x -b "DC=DOMAIN,DC=TLD" -s sub "(&(objectclass=user))" | grep sAMAccountName: | cut -f2 -d" "` to pull usernames via LDAP.
5. **LDAP Anonymous Bind — windapsearch:** Run `./windapsearch.py --dc-ip <DC_IP> -u "" -U` for a cleaner output of all AD users.
6. **Kerbrute (no creds needed):** Run `kerbrute userenum -d <DOMAIN> --dc <DC_IP> /opt/jsmith.txt` to validate usernames via Kerberos Pre-Authentication against a wordlist like `jsmith.txt` (48,705 entries in `flast` format).
7. **Credentialed — CrackMapExec:** With valid creds, run `crackmapexec smb <DC_IP> -u <USER> -p <PASS> --users` for a complete user list with badpwdcount data.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: How many valid usernames can we enumerate with Kerbrute and /opt/jsmith.txt from an unauthenticated standpoint? | **56** | Kerbrute userenum against DC (see walkthrough below) |

### Q1 Walkthrough

**Step 1 — SSH into the attack box:**
```
ssh htb-student@10.129.17.46
# Password: HTB_@cademy_stdnt!
```

**Step 2 — Run Kerbrute username enumeration:**
```
kerbrute userenum -d inlanefreight.local --dc 172.16.5.5 /opt/jsmith.txt
```
```
2024/12/03 12:22:03 >  [+] VALID USERNAME:       tjohnson@inlanefreight.local
2024/12/03 12:22:03 >  [+] VALID USERNAME:       jjones@inlanefreight.local
2024/12/03 12:22:03 >  [+] VALID USERNAME:       jwilson@inlanefreight.local
...SNIP...
2024/12/03 12:22:12 >  [+] VALID USERNAME:       whouse@inlanefreight.local
2024/12/03 12:22:12 >  [+] VALID USERNAME:       emercer@inlanefreight.local
2024/12/03 12:22:13 >  [+] VALID USERNAME:       wshepherd@inlanefreight.local
2024/12/03 12:22:19 >  Done! Tested 48705 usernames (56 valid) in 15.577 seconds  ← ANSWER
```

## Key Takeaways
- Kerbrute is the stealthiest unauthenticated enumeration method — it does not trigger event ID 4625 (logon failure), only event ID 4768 (TGT request), which requires Kerberos event logging to be enabled via Group Policy.
- However, once you switch from enumeration to spraying with Kerbrute, failed Pre-Authentication attempts DO count toward the lockout threshold — stealth only applies to the username enumeration phase.
- CrackMapExec's `--users` flag shows `badpwdcount` per account — always check this before spraying to exclude accounts already near lockout.
- The `badpwdcount` is maintained separately on each DC. For an accurate total, query the DC with the PDC Emulator FSMO role or sum across all DCs.
- The `jsmith.txt` wordlist uses the `flast` (first-initial + lastname) naming convention with ~48,705 entries — the statistically-likely-usernames GitHub repo has other formats too.
- Always log your spray activities: accounts targeted, DC used, time/date, and passwords attempted. If accounts get locked, you need to show your client exactly what you did.

## Gotchas
- Computer accounts (ending with `$`) will show up in ldapsearch results — filter them out before spraying.
- If SMB NULL sessions and LDAP anonymous binds both fail, Kerbrute with a good wordlist is your fallback, but it only finds usernames matching the wordlist's naming convention — you'll miss non-standard usernames.
- `linkedin2username` is a last-resort option for external assessments when no other enumeration method works — generates candidate usernames from a company's LinkedIn page.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[09-enumerating-password-policies]] | [[11-internal-password-spraying-linux]] →
<!-- AUTO-LINKS-END -->
