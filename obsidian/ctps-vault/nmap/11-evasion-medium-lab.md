# LAB — Firewall/IDS Evasion (Medium)

## ID
11

## Module
Nmap

## Kind
lab

## Title
Section 11 — Firewall and IDS/IPS Evasion · Medium Lab

## Description
Medium lab: filtering is tighter, must use UDP. Find the DNS server version of the target via NSE script `dns-nsid`.

## Tags
nmap, lab, evasion, dns, udp, nsid, nse, htb

## Commands
- `sudo nmap -Pn --disable-arp-ping -p53 -sU -sC <IP>`
- `dig CH TXT version.bind @<IP>`

## Scenario
After improvements, the firewall now filters more aggressively. Note: UDP must be used on the VPN. Goal: extract the target's DNS server version.

## Approach
1. **Force UDP scan** with `-sU` (DNS lives on UDP/53).
2. **Skip host discovery** (`-Pn`) and ARP ping (`--disable-arp-ping`) — the filter blocks ICMP.
3. **Use default scripts** (`-sC`) to auto-run `dns-nsid` against UDP/53. That script issues the `version.bind` CHAOS query and prints the version banner.

```bash
sudo nmap -Pn --disable-arp-ping -p53 -sU -sC <IP>
```

Result excerpt:
```
PORT   STATE SERVICE
53/udp open  domain
| dns-nsid:
|_  bind.version: HTB{GoTtgUnyze9Psw4vGjcuMpHRp}
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — DNS server version | **`HTB{GoTtgUnyze9Psw4vGjcuMpHRp}`** | `dns-nsid` NSE script (auto-fired by `-sC`) on UDP/53 returned the flag in `bind.version` |

## Key Takeaways
- `-sC` over UDP/53 is enough — `dns-nsid` runs by default and dumps `version.bind`.
- The DNS `version.bind` query (CHAOS class TXT) is the canonical way to fingerprint a DNS server.
- Manual equivalent: `dig CH TXT version.bind @<IP>`.

## Gotchas
- Without `-Pn`, Nmap will try TCP-based host discovery first — which is filtered → "host seems down".
- `-sU` alone (no `-sC`) only confirms the port is open; you need scripts to extract the banner.
- UDP scans are slow even with one port — be patient.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|nmap]]
← [[10-evasion-easy-lab]] | [[12-evasion-hard-lab]] →
<!-- AUTO-LINKS-END -->
