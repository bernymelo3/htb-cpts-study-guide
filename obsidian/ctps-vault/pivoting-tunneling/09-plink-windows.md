# NOTE — SSH for Windows: plink.exe

## ID
608

## Module
Pivoting, Tunneling, and Port Forwarding

## Kind
notes

## Title
Section 9 — SSH for Windows: plink.exe

## Description
Use Plink (PuTTY Link) on Windows to create SSH tunnels for pivoting — dynamic SOCKS proxies (`-D`) and local port forwards (`-L`) — as an alternative to the Linux `ssh` client.

## Tags
plink, windows, ssh, tunneling, pivoting, socks-proxy

## Commands
- `plink -ssh -D 9050 ubuntu@10.129.x.x`
- `plink -ssh -L 3389:172.16.5.19:3389 ubuntu@10.129.x.x`
- `plink -ssh -L 3389:172.16.5.19:3389 ubuntu@10.129.x.x -pw "HTB_@cademy_stdnt!"`
- `ssh -D 9050 ubuntu@10.129.x.x`
- `ssh -L 3389:172.16.5.19:3389 ubuntu@10.129.x.x`
- `proxychains xfreerdp /u:victor /p:'pass@123' /v:172.16.5.19`

## What This Section Covers
Before Windows 10 Fall 2018, there was no native SSH client. Plink (part of PuTTY) fills that gap, allowing Windows attack hosts or compromised Windows machines to create SSH tunnels. This section explains when to use Plink (living off the land), how to set up dynamic (`-D`) and local (`-L`) port forwards, and how to use Proxifier on Windows to route all traffic (including RDP) through the SOCKS proxy. Linux equivalents are provided for comparison.

## Methodology

### 1. Dynamic Port Forward (SOCKS Proxy) – Flexible pivoting
- On Windows attack host:  
  `plink -ssh -D 9050 ubuntu@<pivot_ip>`
- Keeps a SOCKS4 proxy on `localhost:9050`.
- Use with **Proxifier** (Windows) to tunnel any application (RDP, browser, etc.) through the pivot.
- Linux equivalent: `ssh -D 9050 ubuntu@<pivot_ip>` + `proxychains`.

### 2. Local Port Forward – Single service
- On Windows attack host:  
  `plink -ssh -L 3389:172.16.5.19:3389 ubuntu@<pivot_ip>`
- Forward local port 3389 to internal target’s RDP port.
- Connect RDP to `127.0.0.1:3389` (or `localhost:3389`).
- Linux equivalent: `ssh -L 3389:172.16.5.19:3389 ubuntu@<pivot_ip>`.

### 3. Using Proxifier (Windows)
- Set up dynamic forward (`-D 9050`).
- Open Proxifier → Add proxy server: `127.0.0.1:9050`, SOCKS4.
- Save profile. Launch `mstsc.exe` and connect directly to `172.16.5.19` – Proxifier routes through the tunnel.

### 4. Inline Password
- Add `-pw "password"` to avoid interactive prompts (less secure, but useful for automation).
- Example:  
  `plink -ssh -L 3389:172.16.5.19:3389 ubuntu@10.129.x.x -pw "HTB_@cademy_stdnt!"`

## Multi-step Workflow (Lab Example)

```cmd
# On Windows attack host: create SOCKS proxy
plink -ssh -D 9050 ubuntu@10.129.x.x -pw "password"

# Keep this terminal open. In Proxifier, configure 127.0.0.1:9050 SOCKS4.

# Now RDP directly to internal target (172.16.5.19) using victor:pass@123
mstsc.exe /v:172.16.5.19

# Alternative: local port forward for RDP only
plink -ssh -L 33389:172.16.5.19:3389 ubuntu@10.129.x.x -pw "password"
# Then RDP to 127.0.0.1:33389

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|pivoting-tunneling]]  
← [[07-socat-redirection]] | [[10-sshuttle]] →
<!-- AUTO-LINKS-END -->
