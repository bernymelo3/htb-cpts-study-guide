# NOTE — Host Discovery

## ID
3

## Module
Nmap

## Kind
notes

## Title
Section 3 — Host Discovery

## Description
Detect which hosts are alive on a network without scanning ports. Covers `-sn` (no port scan), ARP vs ICMP echo, and reading TTL to fingerprint OS.

## Tags
nmap, host-discovery, ping-sweep, arp, icmp, ttl

## Commands
- `sudo nmap 10.129.2.0/24 -sn -oA tnet | grep for | cut -d" " -f5`
- `sudo nmap -sn -oA tnet -iL hosts.lst | grep for | cut -d" " -f5`
- `sudo nmap -sn -oA tnet 10.129.2.18 10.129.2.19 10.129.2.20`
- `sudo nmap -sn -oA tnet 10.129.2.18-20`
- `sudo nmap 10.129.2.18 -sn -oA host -PE --packet-trace`
- `sudo nmap 10.129.2.18 -sn -oA host -PE --reason --disable-arp-ping`

## What This Section Covers
Before port-scanning a network, find out which hosts are up. Most reliable signal is ICMP echo, but on the local link Nmap actually uses ARP first because it's faster and harder to firewall. Save every scan with `-oA <name>` for later comparison.

## Methodology
1. **Scan the network range** with `-sn` to disable port scan.
2. **Use `-iL <file>`** when given a host list (common in white-box engagements).
3. **Trace the packets** (`--packet-trace`) to see what Nmap *actually* sent — often it's ARP, not ICMP, on a local subnet.
4. **Force ICMP echo** with `-PE` and **disable ARP** with `--disable-arp-ping` when you need to test ICMP filtering specifically.
5. **Use `--reason`** to learn *why* Nmap marked a host alive (e.g. `arp-response`, `echo-reply`).

## Common Flags
| Flag | Meaning |
|------|---------|
| `-sn` | Ping scan only — no port scan |
| `-oA <basename>` | Save in all 3 formats (.nmap, .gnmap, .xml) |
| `-iL <file>` | Read targets from file |
| `-PE` | ICMP Echo Request ping |
| `--disable-arp-ping` | Skip ARP, force IP-layer probes |
| `--packet-trace` | Show every packet sent/received |
| `--reason` | Show why a port/host got its state |

## Quick Snippet: One-liner to extract live hosts
```bash
sudo nmap 10.129.2.0/24 -sn -oA tnet | grep for | cut -d" " -f5
```

## OS Fingerprinting from TTL (Echo Reply)
The TTL value in the ICMP echo reply is a quick OS hint:
| TTL (received) | Likely OS |
|----------------|-----------|
| ~64 | Linux / Unix |
| ~128 | Windows |
| ~255 | Cisco / network devices |

The TTL is decremented by every hop — so if you see 124, it's `128 - 4` and the host is Windows 4 hops away.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — OS based on the last result | **Windows** | `--packet-trace` showed ICMP echo reply with `ttl=128`, which is the Windows default TTL |

## Key Takeaways
- On the same L2 segment, Nmap will ARP-ping by default — even when you ask for `-PE`. Use `--disable-arp-ping` to test ICMP filtering.
- Always `-oA` your output, every time. You'll need diff between scans.
- TTL is a free OS hint — read it from `--packet-trace`.

## Gotchas
- A "down" host with default `-sn` may be alive but blocking ICMP. Re-test with TCP-SYN ping (`-PS22,80,443`) before writing it off.
- If you forget `sudo`, Nmap silently downgrades probes — host discovery becomes much less reliable.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|nmap]]
← [[02-introduction-to-nmap]] | [[04-host-and-port-scanning]] →
<!-- AUTO-LINKS-END -->
