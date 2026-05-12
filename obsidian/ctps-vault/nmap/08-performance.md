# NOTE — Performance

## ID
8

## Module
Nmap

## Kind
notes

## Title
Section 8 — Performance Tuning

## Description
Make scans faster (or stealthier) with timing templates `-T0..-T5`, RTT timeouts, retry counts, and minimum packet rate.

## Tags
nmap, performance, timing-templates, rtt, max-retries, min-rate

## Commands
- `sudo nmap <CIDR> -F`
- `sudo nmap <CIDR> -F --initial-rtt-timeout 50ms --max-rtt-timeout 100ms`
- `sudo nmap <CIDR> -F --max-retries 0`
- `sudo nmap <CIDR> -F --min-rate 300`
- `sudo nmap <CIDR> -F -T 5`
- `sudo nmap <IP> -p- --min-rate 1000 -T4`

## Knobs You Can Turn
| Flag | What It Controls | Notes |
|------|------------------|-------|
| `-T 0..5` | Timing template (paranoid → insane) | Default `-T 3`. `-T 4` is most common for speed without losing accuracy. |
| `--min-rate <n>` | Min packets/sec | Hard floor — Nmap will push to maintain this. |
| `--max-rate <n>` | Max packets/sec | Useful to be polite on shared networks. |
| `--max-retries <n>` | Retries on no-response | Default 10. Drop to 1 on filtered networks to cut scan time massively. |
| `--initial-rtt-timeout <t>` | Starting RTT | Default 1s. Tune lower on fast LAN (50ms). |
| `--max-rtt-timeout <t>` | Max RTT | Caps wait time per probe. |
| `--min-parallelism <n>` | Min concurrent probes | Forces a parallelism floor. |

## Timing Templates Cheat Sheet
| Template | Name | Use When |
|----------|------|----------|
| `-T0` | paranoid | Extreme IDS evasion (1 probe / 5 min) |
| `-T1` | sneaky | IDS evasion |
| `-T2` | polite | Avoid bandwidth/host stress |
| `-T3` | normal | Default |
| `-T4` | aggressive | Reliable fast LAN — recommended baseline |
| `-T5` | insane | Very fast, may drop accuracy |

Full template values: https://nmap.org/book/performance-timing-templates.html

## Common Speed Recipes
```bash
# Fast TCP enum on a friendly network
sudo nmap -p- --min-rate 1000 -T4 <IP>

# Fast top-100 sweep across a /24
sudo nmap <CIDR> -F --min-rate 300 -T4

# Skip retries to plough through filtered hosts
sudo nmap <CIDR> -F --max-retries 0
```

## Performance vs Accuracy Trade-off
- Cutting `--initial-rtt-timeout` too low → slow hosts get marked as down.
- `--max-retries 0` → filtered ports are invisible; you'll miss real services.
- `-T5` on a flaky network → packets drop, ports vanish from output.

The lab examples show small but real losses — scanning `/24` at default vs `--initial-rtt-timeout 50ms --max-rtt-timeout 100ms` cut runtime ~70% but missed 2 hosts.

## Key Takeaways
- For black-box engagements: start `-T3`, fall back to `-T4` if you trust the network.
- For white-box (whitelisted): crank `--min-rate` to 1000+ and `--max-retries 1`.
- For stealth / IDS evasion: `-T1` or `-T2`, plus the techniques in the next section.
- Always verify "missing" hosts with a slower second pass — speed kills accuracy.

## Gotchas
- `-T5` can DOS small targets (low-end IoT, embedded HTTP servers). Tone down for fragile hosts.
- Many "gone offline" hosts during a fast scan are still alive — they just didn't reply in time.
- `--min-rate` ignores RTT — you can flood a slow link before getting any response back.
- Templates *override* manual settings if specified after them. Order matters: put `-T 4` first, then your overrides.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|nmap]]
← [[07-nmap-scripting-engine]] | [[09-firewall-ids-ips-evasion]] →
<!-- AUTO-LINKS-END -->
