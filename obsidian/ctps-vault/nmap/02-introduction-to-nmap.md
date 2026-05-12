# NOTE — Introduction to Nmap

## ID
2

## Module
Nmap

## Kind
notes

## Title
Section 2 — Introduction to Nmap

## Description
Overview of Nmap as a scanner: what it does, the syntax skeleton, and the core scan techniques (TCP-SYN, Connect, UDP, NULL/FIN/Xmas, Idle, etc.).

## Tags
nmap, scanning, syn-scan, tcp-scan, udp-scan, reference, theory

## Commands
- `nmap <scan types> <options> <target>`
- `nmap --help`
- `sudo nmap -sS localhost`

## What This Section Covers
Nmap (Network Mapper) is an open-source network analysis & auditing tool. It does host discovery, port scanning, service/version detection, OS detection, and runs Lua-based NSE scripts. Default scan when run as root is `-sS` (TCP-SYN). Without root it falls back to `-sT` (TCP Connect).

## Syntax Skeleton
```
nmap <scan types> <options> <target>
```

## Scan Techniques (from `nmap --help`)
| Flag | Scan Type |
|------|-----------|
| `-sS` | TCP SYN scan — half-open, fast, default for root |
| `-sT` | TCP Connect — full 3-way handshake, default for non-root |
| `-sA` | TCP ACK — for firewall rule mapping |
| `-sW` | TCP Window |
| `-sM` | TCP Maimon |
| `-sU` | UDP scan |
| `-sN` / `-sF` / `-sX` | TCP NULL / FIN / Xmas |
| `--scanflags <flags>` | Custom TCP flag combos |
| `-sI <zombie[:port]>` | Idle scan (uses 3rd-party host as proxy) |
| `-sY` / `-sZ` | SCTP INIT / COOKIE-ECHO |
| `-sO` | IP protocol scan |
| `-b <FTP relay>` | FTP bounce scan |

## TCP-SYN Scan Behaviour
The half-open scan is the workhorse:
- Send `SYN` only — never complete the 3-way handshake.
- Target replies with `SYN-ACK` → port is **open**.
- Target replies with `RST` → port is **closed**.
- No reply → port is **filtered** (firewall dropping).

```bash
sudo nmap -sS localhost
```

```
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
5432/tcp open  postgresql
5901/tcp open  vnc-1
```

## Key Takeaways
- `-sS` requires root because it crafts raw packets.
- Output gives port number → state → service guess (from `nmap-services`, not actually probed unless `-sV` is used).
- Without `-sV`, the service name is a *guess based on the port number*, not the real service.

## Gotchas
- `-sT` is louder than `-sS` — it completes a full TCP connection and gets logged everywhere.
- Service column ≠ confirmed service. Port 80 listed as `http` could be running anything.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|nmap]]
← [[01-enumeration]] | [[03-host-discovery]] →
<!-- AUTO-LINKS-END -->
