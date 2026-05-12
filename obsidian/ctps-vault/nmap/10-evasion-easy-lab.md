# LAB — Firewall/IDS Evasion (Easy)

## ID
10

## Module
Nmap

## Kind
lab

## Title
Section 10 — Firewall and IDS/IPS Evasion · Easy Lab

## Description
Easy evasion lab: identify the target OS while staying under the IDS alert threshold. Status page at `http://<target>/status.php` tracks alerts.

## Tags
nmap, lab, evasion, ids, os-fingerprinting, htb

## Commands
- `sudo nmap -sV --top-ports 10 --disable-arp-ping <IP>`
- `curl http://<IP>/status.php` — watch alert counter

## Scenario
Client wants their IDS/IPS tested. The status page shows alert counts. Hit the threshold and you're banned. Identify the OS without tripping it.

## Approach
1. **Quiet scan only the top ports** — `--top-ports 10` is enough to fingerprint the OS via service banners.
2. **Disable ARP ping** with `--disable-arp-ping` to avoid an extra discovery probe.
3. **Use `-sV`** — service banners give the OS for free (e.g. `OpenSSH 7.2p2 Ubuntu 4ubuntu2.10` → Ubuntu).

```bash
sudo nmap -sV --top-ports 10 --disable-arp-ping <IP>
```

Result excerpt:
```
22/tcp  open  ssh   OpenSSH 7.2p2 Ubuntu 4ubuntu2.10 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http  Apache httpd 2.4.18 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — OS of provided machine | **Ubuntu** | `-sV` on top ports → SSH/Apache banners both end in `(Ubuntu)` |

## Key Takeaways
- Service version banners contain the distro name → no need for `-O` (which is loud).
- Top-10 port scan + `-sV` is enough OS fingerprinting for most Linux targets.
- `--disable-arp-ping` removes one less discovery probe — small but matters when alerts are counted.

## Gotchas
- Using `-O` (OS scan) here would burn many alerts for the same answer — version banners give it cheaply.
- The `Service Info: OS: Linux` line is generic; the *distro* (Ubuntu) only appears in individual service banners.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|nmap]]
← [[09-firewall-ids-ips-evasion]] | [[11-evasion-medium-lab]] →
<!-- AUTO-LINKS-END -->
