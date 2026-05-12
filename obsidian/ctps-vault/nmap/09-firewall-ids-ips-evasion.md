# NOTE — Firewall and IDS/IPS Evasion

## ID
9

## Module
Nmap

## Kind
notes

## Title
Section 9 — Firewall and IDS/IPS Evasion

## Description
Detect and bypass firewalls + IDS/IPS using ACK scans, decoys (`-D RND:n`), source IP spoofing (`-S`), source-port abuse (`--source-port 53`), and DNS proxying.

## Tags
nmap, evasion, firewall, ids, ips, decoy, source-port, ack-scan

## Commands
- `sudo nmap <IP> -p 21,22,25 -sS --packet-trace`
- `sudo nmap <IP> -p 21,22,25 -sA --packet-trace`
- `sudo nmap <IP> -p 80 -sS -D RND:5`
- `sudo nmap <IP> -p 445 -O -S <spoofed_ip> -e tun0`
- `sudo nmap <IP> -p 50000 -sS --source-port 53`
- `ncat -nv --source-port 53 <IP> 50000`
- `nmap --dns-servers <ns1>,<ns2> <IP>`

## Concepts
**Firewall**: stateful packet filter — applies allow/deny rules per connection.
**IDS**: passive — watches traffic and *alerts* on bad patterns.
**IPS**: active — IDS + automatic blocking.

Evasion goal: get probes through filters *and* under detection thresholds. If the IPS counts your alerts and bans you after N, you must reduce per-IP signal.

## How Filtered Ports Get Filtered
| Action | Network signal |
|--------|---------------|
| Drop (silent) | No reply at all → Nmap retries `--max-retries` times → marks `filtered` |
| Reject | ICMP type 3 / code 3 (port unreachable) → marks `filtered` quickly |
| TCP reset | RST flag → marks `closed` (port is filtered but firewall replies "closed") |

## ACK Scan (`-sA`) — Firewall Mapping
The ACK scan doesn't tell you *open vs closed*; it tells you *filtered vs unfiltered*. Stateful firewalls usually let ACK packets through (because they can't tell direction) — so it's a way to map which ports the firewall has rules on.

```bash
sudo nmap <IP> -p 21,22,25 -sA -Pn -n --disable-arp-ping --packet-trace
```
- `RCVD ... R` → port is `unfiltered` (firewall let it through, host replied with RST).
- No reply / ICMP error → `filtered`.

## Detecting IDS/IPS
You can't fingerprint IDS directly, but you can probe behaviour:
- **Use multiple VPS source IPs.** If one gets blocked while another doesn't after the same scan, IPS is active.
- **Aggressively scan one port and watch.** Sudden silence → you tripped a threshold.
- Recommendation: rotate VPS, scan slowly, and never use your real IP for noisy probes.

## Decoys (`-D`)
Mix your real IP into a list of fake source IPs so logs show many origins.
```bash
sudo nmap <IP> -p 80 -sS -D RND:5
```
- `RND:5` = generate 5 random IPs.
- Or specify manually: `-D 1.2.3.4,5.6.7.8,ME,9.10.11.12` (`ME` = your IP).
- Decoys must look alive — if the firewall does anti-spoofing checks (BCP38), decoys get filtered out.

## Source IP Spoofing (`-S`)
If a service is firewalled by source-subnet, try spoofing a source the firewall trusts:
```bash
sudo nmap <IP> -n -Pn -p 445 -O -S 10.129.2.200 -e tun0
```
- `-S` sets a fake source IP (you won't get replies unless you can sniff for them).
- `-e <iface>` forces the interface (needed when you spoof, otherwise routing breaks).

This is also useful when the network only exposes a service to internal IP ranges and you can spoof an internal address.

## Source Port Abuse (`--source-port` / `-g`)
Many firewalls trust source port **53/UDP** (DNS) and sometimes **53/TCP**, **80**, **443** because of legacy rules.
```bash
# A filtered port becomes open when source port is 53:
sudo nmap <IP> -p 50000 -sS --source-port 53
```
Same trick with `nc`/`ncat` to actually connect:
```bash
ncat -nv --source-port 53 <IP> 50000
```

## DNS Proxying (`--dns-servers`)
Force Nmap to resolve hostnames via a specific DNS server. Useful when you're inside a DMZ and the corporate DNS is more permissive than the public one.
```bash
nmap --dns-servers 10.10.10.1,10.10.10.2 <internal_target>
```

## Quick Reference: Common Evasion Stack
```bash
# Stealth + spoofed source port
sudo nmap <IP> -p- -sS --source-port 53 -T2 -Pn -n --disable-arp-ping --max-retries 1
```
- `-T2` polite timing
- `-Pn -n` no host discovery, no DNS
- `--disable-arp-ping` skip ARP
- `--source-port 53` masquerade as DNS

## Key Takeaways
- ACK scan = firewall map, not port enumeration.
- `--source-port 53` works far more often than it should — try it first when ports are mysteriously filtered.
- Decoys help with logs but rarely with stateful IDS/IPS.
- Use VPS rotation when you suspect IPS auto-block — losing one IP shouldn't block your engagement.

## Gotchas
- Spoofed source IPs (`-S`) are usually filtered by upstream BCP38 — don't expect packets back unless you control the path.
- Decoys make scans slower (you send N times the packets).
- IDS counts alerts even when scans "succeed" — the lab scenarios deduct points if you trip too many.
- `--source-port` doesn't change source IP; the firewall still sees your real address.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|nmap]]
← [[08-performance]] | [[10-evasion-easy-lab]] →
<!-- AUTO-LINKS-END -->
