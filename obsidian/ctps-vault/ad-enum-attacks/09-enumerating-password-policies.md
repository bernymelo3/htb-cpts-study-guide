## ID
900

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 9 — Enumerating & Retrieving Password Policies

## Description
Covers multiple methods (credentialed and unauthenticated) to enumerate domain password policies from Linux and Windows, which is a prerequisite step before launching password spraying attacks.

## Tags
active-directory, password-policy, smb-null-session, ldap-anonymous-bind, enum4linux, password-spraying

## Commands
- crackmapexec smb <DC_IP> -u <USER> -p <PASS> --pass-pol
- rpcclient -U "" -N <DC_IP>
- rpcclient $> getdompwinfo
- enum4linux -P <DC_IP>
- enum4linux-ng -P <DC_IP> -oA <OUTPUT_PREFIX>
- ldapsearch -h <DC_IP> -x -b "DC=<DOMAIN>,DC=<TLD>" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength
- net use \\<DC_HOSTNAME>\ipc$ "" /u:""
- net accounts
- import-module .\PowerView.ps1; Get-DomainPolicy

## What This Section Covers
Before password spraying, you must know the domain's password policy to avoid locking out accounts. This section teaches how to retrieve the policy using credentialed tools (CrackMapExec, rpcclient, net.exe, PowerView) and unauthenticated techniques (SMB NULL sessions, LDAP anonymous binds). Understanding the lockout threshold, lockout duration, and minimum password length dictates how aggressively you can spray.

## Methodology
1. **Credentialed (Linux):** Use `crackmapexec smb <DC_IP> -u <USER> -p <PASS> --pass-pol` to dump the full password policy if you already have valid creds.
2. **SMB NULL Session (Linux):** Connect with `rpcclient -U "" -N <DC_IP>`, then run `querydominfo` to confirm NULL session access and `getdompwinfo` to pull the policy.
3. **enum4linux / enum4linux-ng (Linux):** Run `enum4linux -P <DC_IP>` or `enum4linux-ng -P <DC_IP> -oA <PREFIX>` for cleaner output with JSON/YAML export.
4. **LDAP Anonymous Bind (Linux):** Use `ldapsearch -h <DC_IP> -x -b "DC=DOMAIN,DC=TLD" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength` to extract the policy via LDAP.
5. **NULL Session (Windows):** Establish with `net use \\DC01\ipc$ "" /u:""` to confirm unauthenticated access.
6. **Credentialed (Windows):** Run `net accounts` for a quick summary, or `Get-DomainPolicy` from PowerView for full detail including complexity flags and Kerberos policy.
7. **Analyze the policy:** Determine safe spray parameters — with lockout threshold of 5, attempt 2–3 passwords per round, wait 31+ minutes between rounds.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: What is the default Minimum password length when a new domain is created? | **7** | Default domain policy table in the module text |
| Q2: What is the minPwdLength set to in the INLANEFREIGHT.LOCAL domain? | **8** | LDAP anonymous bind against DC at 172.16.5.5 (see walkthrough below) |

### Q2 Walkthrough

**Step 1 — SSH into the attack box:**
```
ssh htb-student@10.129.17.46
# Password: HTB_@cademy_stdnt!
```

**Step 2 — Ping sweep to discover live hosts:**
```
for i in {1..254}; do (ping -c 1 172.16.5.$i | grep "bytes from" &); done
```
```
64 bytes from 172.16.5.5: icmp_seq=1 ttl=128 time=2.82 ms    ← DC (TTL 128 = Windows)
64 bytes from 172.16.5.225: icmp_seq=1 ttl=64 time=0.051 ms   ← attack host (TTL 64 = Linux)
```

**Step 3 — Pull the password policy via LDAP anonymous bind:**
```
ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength
```
```
forceLogoff: -9223372036854775808
lockoutDuration: -18000000000
lockOutObservationWindow: -18000000000
lockoutThreshold: 5
maxPwdAge: -9223372036854775808
minPwdAge: -864000000000
minPwdLength: 8              ← ANSWER
modifiedCountAtLastProm: 0
nextRid: 1002
pwdProperties: 1
pwdHistoryLength: 24
```

## Key Takeaways
- Always retrieve the password policy before spraying — it determines your safe spray cadence and candidate passwords.
- SMB NULL sessions and LDAP anonymous binds are legacy misconfigs (pre-Windows Server 2003) but still appear in real environments due to in-place DC upgrades.
- Windows password complexity (3/4 of upper, lower, number, special) is weak — `Password1` and `Welcome1` both satisfy it, making them prime spray candidates against an 8-char minimum.
- The default AD domain policy has no lockout threshold (0), meaning unlimited failed attempts — many orgs never change this.
- enum4linux-ng is preferred over enum4linux for its JSON/YAML export (`-oA`), which is useful for scripting and feeding into other tools.
- If you cannot retrieve the policy (e.g., external assessment), limit to one spray attempt max, wait over an hour, or ask the client directly.

## Gotchas
- `ldapsearch -h` is deprecated in newer versions — use `-H ldap://<DC_IP>` instead.
- The raw LDAP time values are negative 100-nanosecond intervals (e.g., `-18000000000` = 30 minutes) — don't confuse them with the human-readable fields like `minPwdLength` which are plain integers.
- `net use` error codes from Windows reveal account state: 1331 = disabled, 1326 = wrong password, 1909 = locked out — useful for confirming policy enforcement during testing.
- Some orgs require manual admin unlock (not auto-unlock after 30 min) — locking out accounts in that scenario is a much bigger problem.
- The tool/port mapping matters: rpcclient uses 135/TCP, smbclient uses 445/TCP, nmblookup uses 137/UDP — know which ports need to be open for each approach.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[08-password-spraying-overview]] | [[10-password-spraying-user-list]] →
<!-- AUTO-LINKS-END -->
