# LAB — Firewall/IDS Evasion (Hard)

## ID
12

## Module
Nmap

## Kind
lab

## Title
Section 12 — Firewall and IDS/IPS Evasion · Hard Lab

## Description
Hard lab: admin trained on IDS/IPS, services moved around. Identify a service version on a non-default port using source-port evasion.

## Tags
nmap, lab, evasion, source-port, ibm-db2, ftp, banner, htb

## Commands
- `sudo nmap -g53 --max-retries=1 -Pn -p- --disable-arp-ping <IP>`
- `sudo nc -nv -s <PWNIP> -p53 <IP> 50000`

## Scenario
Tighter filtering, services on non-standard ports. Goal: extract the version banner of the running service.

## Approach
1. **Full TCP scan with source port spoofing.** Use `-g53` (alias for `--source-port 53`) — DNS source port often bypasses lazy firewall rules.
2. **Skip ICMP** (`-Pn`) and ARP ping. Drop retries to 1 to keep noise low.
3. **Connect with source port 53** using `nc` to grab the banner from the discovered service.

### Step 1 — full port scan with source-port evasion
```bash
sudo nmap -g53 --max-retries=1 -Pn -p- --disable-arp-ping <IP>
```
Result:
```
22/tcp    open  ssh
80/tcp    open  http
50000/tcp open  ibm-db2
```
Port `50000` is interesting (Nmap labels it `ibm-db2` from the services file but it could be anything).

### Step 2 — banner-grab with matching source port
```bash
sudo nc -nv -s <PWNIP> -p53 <IP> 50000
# 220 HTB{kjnsdf2n982n1827eh76238s98di1w6}
```
The `220` reply identifies it as **ProFTPd** running on port 50000 — the firewall only lets traffic through when source port is 53.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Service version flag | **`HTB{kjnsdf2n982n1827eh76238s98di1w6}`** | Source-port-53 scan revealed `50000/tcp open`; `nc -p53 <IP> 50000` returned the FTP `220` banner |

## Key Takeaways
- `-g53` = `--source-port 53`. Same evasion technique, shorter syntax.
- Once Nmap finds an open port behind source-port filtering, your *follow-up* connection (`nc`, `curl`, etc.) must also use the same source port. Otherwise the firewall blocks it.
- `nc -s <my_ip> -p <src_port>` is how you bind to a specific source IP+port. Requires root because privileged source port.
- Non-standard ports (50000, 31337, 8443) often hide the most interesting services on hardened targets.

## Gotchas
- Without `-p-`, you'd miss port 50000 (top-1000 doesn't include it).
- `--max-retries 1` speeds up the scan but can hide services on a flaky link.
- The Nmap "service" column is just a guess from `nmap-services` — it said `ibm-db2` but the actual service was ProFTPd. Always confirm with a real banner grab.
- Source port 53 (TCP) is bound on most Linux distros — you may need `sudo` for nc and ensure no resolver is using it locally.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|nmap]]
← [[11-evasion-medium-lab]]
<!-- AUTO-LINKS-END -->
