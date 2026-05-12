# NOTE — Host and Port Scanning

## ID
4

## Module
Nmap

## Kind
notes

## Title
Section 4 — Host and Port Scanning

## Description
Covers TCP and UDP port scanning, the six possible port states, the difference between SYN / Connect / ACK scans, and how to interpret `filtered` results to fingerprint firewall behaviour.

## Tags
nmap, port-scanning, syn-scan, connect-scan, udp-scan, firewall, reference

## Commands
- `sudo nmap 10.129.2.28 --top-ports=10`
- `sudo nmap --open -p- <IP> -T5`
- `sudo nmap -p 22,25,80,139,445 <IP>`
- `sudo nmap -p- <IP>` — all 65535 TCP ports
- `sudo nmap -F <IP>` — top 100 ports
- `sudo nmap <IP> -p 21 --packet-trace -Pn -n --disable-arp-ping`
- `sudo nmap -p 443 -sT --reason <IP>`
- `sudo nmap <IP> -F -sU` — UDP top 100
- `sudo nmap -p445 <IP> -sV -T5`

## Port States (Six Possible)
| State | Meaning |
|-------|---------|
| `open` | Connection accepted (TCP / UDP / SCTP). |
| `closed` | RST flag returned — host is alive but nothing listens. |
| `filtered` | No response or ICMP error — firewall is dropping. |
| `unfiltered` | Only with `-sA` — firewall lets the packet through but state unknown. |
| `open\|filtered` | No response — could be firewalled or could be UDP with no banner. |
| `closed\|filtered` | Only with IP ID idle scans. |

## Discovering Open TCP Ports
Default with root: `-sS` (SYN). Without root: `-sT` (Connect). You can target ports several ways:
- One by one: `-p 22,25,80,139,445`
- Range: `-p 22-445`
- Top N from Nmap's frequency DB: `--top-ports=10`
- All 65535: `-p-`
- Top 100 fast: `-F`
- Open only (cleaner output): `--open`

## SYN vs Connect vs ACK
| Scan | Flag | Sends | Stealth | Reliable? | Use Case |
|------|------|-------|---------|-----------|----------|
| SYN | `-sS` | SYN only, no ACK back | Better (half-open) | Yes | Default for root |
| Connect | `-sT` | Full 3-way handshake | Loud (logs everywhere) | Most accurate | Non-root, "polite" scans |
| ACK | `-sA` | ACK only | Specialised | Maps firewalls, doesn't find open ports | Detect stateful filters |

## SYN Scan — Packet Trace
```bash
sudo nmap 10.129.2.28 -p 21 --packet-trace -Pn -n --disable-arp-ping
```
- `SENT ... S` → we sent SYN.
- `RCVD ... RA` → target sent RST+ACK → port **closed**.
- `RCVD ... SA` → target sent SYN+ACK → port **open**.
- No `RCVD` after retries → **filtered**.

## Filtered: dropped vs rejected
- **Dropped** → no response at all. Nmap retries (default `--max-retries 10`). Slow.
- **Rejected** → ICMP type 3 / code 3 (`port unreachable`). Fast response, but still labelled `filtered`. Often a firewall reject rule.

## UDP Scanning
`sudo nmap <IP> -F -sU` — much slower than TCP because UDP is stateless.
- **Open**: app responded. Confirmed open.
- **Closed**: ICMP type 3 / code 3 (port unreachable).
- **`open|filtered`**: no response — could be either. Most common UDP result.

UDP scans need the application itself to respond; if the listener doesn't speak the right probe, you get nothing.

## Service / Version Scan (`-sV`)
```bash
sudo nmap <IP> -p 445 -sV --reason
```
Forces Nmap to actually talk to the port and match the banner against `nmap-service-probes`. This is what gives you `Samba smbd 3.X - 4.X (workgroup: WORKGROUP)` instead of just `microsoft-ds`.

The `Service Info: Host: <hostname>` line is gold — it leaks the **NetBIOS hostname** without you ever logging in.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Total open TCP ports | **7** | `sudo nmap --open -p- <IP> -T5 \| grep "/tcp" \| wc -l` → ports 22, 80, 110, 139, 143, 445, 31337 |
| Q2 — Hostname (case-sensitive) | **NIX-NMAP-DEFAULT** | `sudo nmap -p445 <IP> -T5 -sV` → `Service Info: Host: NIX-NMAP-DEFAULT` |

### Q1 reference command
```bash
sudo nmap --open -p- <IP> -T5 | grep "/tcp" | wc -l
```

### Q2 reference command
```bash
sudo nmap -p445 <IP> -T5 -sV | grep "Host:" | cut -d " " -f3,4
```

## Key Takeaways
- Always run `-p-` once on a fresh target before deciding what to ignore. Top 1000 misses non-standard ports (like 31337 in the lab).
- `-sV` on SMB / NetBIOS leaks the hostname — try it first against port 445 / 139.
- `--reason` separates "no response" (likely dropped) from "ICMP unreachable" (likely rejected) — different firewall postures.

## Gotchas
- `-T5` is fast but unreliable on flaky networks (drops packets). Use `-T4` for "fast but accurate".
- Nmap retries 10× by default. On a filtered port that's 10 unanswered packets per port — `-p-` with default retries can take hours. Drop with `--max-retries 1` if you trust the network.
- `--top-ports=N` uses Nmap's static frequency DB. A target with weird ports (31337, 50000) won't show up unless you run `-p-`.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|nmap]]
← [[03-host-discovery]] | [[05-saving-the-results]] →
<!-- AUTO-LINKS-END -->
