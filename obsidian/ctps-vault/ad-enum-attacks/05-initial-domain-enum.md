# NOTE — Initial Enumeration of the Domain

## ID
704

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 5 — Initial Enumeration of the Domain

## Description
Perform passive and active enumeration of an Active Directory domain to discover live hosts, key services, and valid user accounts using Wireshark, fping, Nmap, and Kerbrute.

## Tags
enumeration, ad-enumeration, nmap, kerbrute, wireshark, host-discovery

## Commands
- `sudo -E wireshark`
- `sudo tcpdump -i ens224 -w capture.pcap`
- `sudo responder -I ens224 -A`
- `fping -asgq 172.16.5.0/23`
- `fping -asgq 172.16.5.0/23 2>/dev/null | head -n -14 > hosts.txt`
- `sudo nmap -v -A -iL hosts.txt -oA /home/htb-student/Documents/host-enum`
- `sudo nmap -sV --open -iL hosts.txt -oA service-scan`
- `kerbrute userenum -d INLANEFREIGHT.LOCAL --dc 172.16.5.5 jsmith.txt -o valid_ad_users`

## What This Section Covers
This section teaches initial enumeration of a Windows domain from an unauthenticated position on the internal network. You learn passive host discovery (Wireshark, tcpdump, Responder -A), active ICMP sweeps with `fping`, full port and service enumeration with Nmap, and stealthy user enumeration using Kerberos pre‑authentication (`kerbrute`). The goal is to build a target list, identify Domain Controllers, legacy hosts, and valid usernames for later attacks.

## Methodology
1. **Passive listening** – capture live network traffic with Wireshark or tcpdump to see ARP, MDNS, and LLMNR/NBT‑NS traffic that reveals hidden hosts.
2. **Responder analyze mode** – run `responder -I <interface> -A` to passively listen for name resolution requests without poisoning.
3. **ICMP sweep** – use `fping -asgq <subnet>` to find all responsive hosts in the scope (`172.16.5.0/23`).
4. **Service enumeration** – run Nmap aggressive scan (`-A`) against the list of live hosts, saving all output (`-oA`).
5. **User enumeration** – use `kerbrute userenum` against the Domain Controller with a statistically‑likely username wordlist (e.g., `jsmith.txt`) to enumerate valid AD users without triggering loud LDAP or SMB queries.

## Multi-step Workflow
```bash
# 1. Passive capture (optional, let it run while scanning)
sudo tcpdump -i eth0 -w capture.pcap

# 2. Analyze mode to spot hostnames
sudo responder -I eth0 -A

# 3. ICMP sweep to find live hosts
fping -asgq 172.16.5.0/23 2>/dev/null > alive.txt


# 4. Aggressive Nmap scan on all alive hosts
sudo nmap -v -A -iL alive.txt -oA host-enum

From the common name it says the Host probabably tells domain controllers
* hostnames 
* domain names 
* certificate authority names
* ADCS presence
* naming conventions
* trust relationships
We caught kerberos 88 ldap 389 global catolog 3268 dns 53

# 5. Targeted user enumeration with kerbrute
kerbrute userenum -d INLANEFREIGHT.LOCAL --dc 172.16.5.5 /opt/wordlists/statistically-likely-usernames/jsmith.txt -o valid_users.txt

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[04-external-recon]] | [[06-llmnr-nbtns-poisoning-linux]] →
<!-- AUTO-LINKS-END -->
