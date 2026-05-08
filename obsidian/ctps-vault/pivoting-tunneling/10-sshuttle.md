# NOTE — SSH Pivoting with Sshuttle

## ID
609

## Module
Pivoting, Tunneling, and Port Forwarding

## Kind
notes

## Title
Section 10 — SSH Pivoting with Sshuttle

## Description
Use sshuttle to create a transparent VPN‑like tunnel over SSH, routing entire subnets through a pivot host so tools work natively without proxychains.

## Tags
sshuttle, pivoting, tunneling, vpn, ssh

## Commands
- `sudo apt-get install sshuttle`
- `sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23 -v`
- `sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.19 -v`
- `sudo sshuttle --dns -r ubuntu@10.129.202.64 172.16.5.0/23 -v`
- `sudo nmap -v -A -sT -p3389 172.16.5.19 -Pn`
- `xfreerdp /v:172.16.5.19 /u:victor /p:'pass@123' /cert:ignore`
- `sed -i '/172.16.5.19/d' ~/.config/freerdp/known_hosts2`
- `sudo iptables -t nat -L`

## What This Section Covers
Sshuttle is a Python tool that creates a transparent TCP‑only VPN over SSH by injecting iptables rules on your attack host. Unlike `ssh -D` + proxychains, you don’t need to prefix commands – you run tools directly against internal IPs. This section explains installation, usage (subnet or single host), limitations (TCP only, requires sudo), and how to handle RDP certificate errors in the lab.

## Methodology
1. **Install sshuttle** – `sudo apt-get install sshuttle`.
2. **Run sshuttle** – specify pivot host (`-r user@ip`) and the internal subnet to route (e.g., `172.16.5.0/23`). Use `-v` for verbose.
3. **Enter pivot’s SSH password** when prompted.
4. **Run tools directly** – nmap with `-sT` (TCP connect), xfreerdp, etc., against internal IPs.
5. **Clean up** – Ctrl+C stops sshuttle and removes iptables rules.

## Multi-step Workflow
```bash
# Install
sudo apt-get install sshuttle

# Route entire /23 subnet through pivot
sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.0/23 -v
# Enter password: HTB_@cademy_stdnt!

# In another terminal, scan internal target (TCP connect scan)
sudo nmap -v -A -sT -p3389 172.16.5.19 -Pn

# RDP directly (no proxychains)
xfreerdp /v:172.16.5.19 /u:victor /p:'pass@123' /cert:ignore

# Optional: route only a single host
sudo sshuttle -r ubuntu@10.129.202.64 172.16.5.19 -v

# With DNS forwarding (if needed)
sudo sshuttle --dns -r ubuntu@10.129.202.64 172.16.5.0/23 -v

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|pivoting-tunneling]]  
← [[09-plink-windows]] | [[11-rpivot]] →
<!-- AUTO-LINKS-END -->
