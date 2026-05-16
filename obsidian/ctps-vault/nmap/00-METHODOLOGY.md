# NOTE — Network Enumeration with Nmap Methodology (Exam Playbook)

## ID
719

## Module
Network Enumeration with Nmap

## Kind
methodology

## Title
Network Enumeration with Nmap — Full Scanning Methodology

## Description
Exam-ready scanning playbook: host discovery → full TCP port scan → service/version enum → NSE deepening → output/reporting, plus the two escape hatches (performance tuning when slow, firewall/IDS evasion when filtered/blocked). Decision-tree first; every command drawn from this vault's own nmap notes.

## Tags
methodology, nmap, exam, cheatsheet, decision-tree, port-scan, host-discovery, service-enum, nse, evasion, source-port, filtered, host-seems-down, slow-scan, udp, banner-grab, ttl

---

## TL;DR — The 7-Phase Flow

1. **Host discovery** — `-sn` over the range → list of live hosts (or `-Pn` to skip if it lies).
2. **Full TCP port scan** — `-p-` on every live host → *all* open ports (never trust top-1000).
3. **Service / version detection** — `-sV -sC` on the open ports only → exact versions + free OS/hostname.
4. **NSE deepening** — `--script vuln` / `http-enum` + **manual `nc` banner grab** for anything odd.
5. **Save everything** — `-oA` reflexively; `xsltproc` XML→HTML for the report.
6. **Too slow?** → performance tuning: `-T4 --min-rate --max-retries`.
7. **Filtered / blocked / alerts counted?** → evasion: `-sA` map, `--source-port 53`, `-Pn -n`, decoys, UDP.

> **Golden rule:** always `-p-` once before deciding what to ignore (the lab's real service was on `31337` / `50000` — top-1000 misses it). Always `-oA` (slow scans are expensive to repeat). The `SERVICE` column is a *guess from `nmap-services`* until you run `-sV` — confirm with a real banner.

> **OPSEC fork (alerts counted — exam/IDS lab):** do **not** run `-O`, `-A`, or `--script vuln` when a status page counts alerts. Service banners give you the distro/OS for free. Use `sudo nmap -sV --top-ports 10 --disable-arp-ping $IP` and read the `(Ubuntu)` etc. out of the SSH/Apache banner. See Gotcha #9.

---

## Phase 1 — Host Discovery

**Goal:** find which hosts are alive before wasting time port-scanning dead IPs.

| Trigger / Precondition | Do this |
|---|---|
| Given a CIDR / range, no host list | `sudo nmap $CIDR -sn -oA tnet \| grep for \| cut -d" " -f5` |
| Given a host list (white-box) | `sudo nmap -sn -oA tnet -iL hosts.lst \| grep for \| cut -d" " -f5` |
| Need to know *why* a host is "up" | add `--reason` (e.g. `arp-response`, `echo-reply`) |
| Need to test ICMP filtering specifically | `sudo nmap $IP -sn -oA host -PE --reason --disable-arp-ping` |
| Want to see what Nmap actually sent | add `--packet-trace` (on a local subnet it ARP-pings even when you ask for `-PE`) |

```bash
sudo nmap $CIDR -sn -oA tnet | grep for | cut -d" " -f5
sudo nmap $IP -sn -oA host -PE --packet-trace
```

**OS hint for free** — read the TTL from the echo reply / packet trace:

| TTL received | Likely OS |
|---|---|
| ~64 | Linux / Unix |
| ~128 | Windows |
| ~255 | Cisco / network device |

(TTL is decremented per hop: `124` = Windows 4 hops away.)

> **Output checkpoint:** you have a list of live IPs + a rough OS guess per host. Saved as `tnet.{nmap,gnmap,xml}`.

---

## Phase 2 — Full TCP Port Scan

**Goal:** every open TCP port on each live host. Run this *before* deciding what's interesting.

| Trigger / Precondition | Do this |
|---|---|
| Fresh target, want everything | `sudo nmap --open -p- $IP -T5` (or `-T4` on flaky links) |
| Quick triage first | `sudo nmap $IP -F` (top 100) or `--top-ports=10` |
| Host discovery said "down" but you suspect it's alive | add `-Pn` (skip ping) — and see Phase 7 |
| Need to know dropped vs rejected | `sudo nmap -p 443 -sT --reason $IP` |
| Want to watch the handshake | `sudo nmap $IP -p 21 --packet-trace -Pn -n --disable-arp-ping` |

```bash
sudo nmap --open -p- $IP -T5
# count open TCP ports:
sudo nmap --open -p- $IP -T5 | grep "/tcp" | wc -l
```

**Six port states** — know what each means:

| State | Meaning |
|---|---|
| `open` | connection accepted |
| `closed` | RST returned — host alive, nothing listening |
| `filtered` | no response / ICMP error — firewall dropping |
| `unfiltered` | `-sA` only — passes filter, state unknown |
| `open\|filtered` | no response — firewalled *or* UDP with no banner |
| `closed\|filtered` | IP ID idle scans only |

`-sS` (SYN, default as root) is stealthier; `-sT` (Connect, default non-root) completes the handshake and gets logged everywhere; `-sA` (ACK) **maps the firewall** — `filtered` vs `unfiltered`, not open vs closed.

> **Output checkpoint:** full list of open TCP ports per host — including non-standard ones (`31337`, `50000`) that top-1000 would have hidden.

---

## Phase 3 — Service / Version Detection

**Goal:** map each open port to an *exact* service + version so you can search exploits/CVEs.

| Trigger / Precondition | Do this |
|---|---|
| Have open ports, need versions | `sudo nmap $IP -p- -sV` (or scope to found ports — much faster) |
| SMB/NetBIOS open (139/445) | `sudo nmap -p445 $IP -sV` → `Service Info: Host:` leaks the NetBIOS hostname |
| `-sV -p-` is slow, want progress | add `--stats-every=5s` or `-v` (prints ports as found); press `[Space]` for live status |
| Want to see the probe exchange | `sudo nmap $IP -p- -sV -Pn -n --disable-arp-ping --packet-trace` |

```bash
sudo nmap $IP -p <OPEN_PORTS> -sV --reason
# hostname leak from SMB:
sudo nmap -p445 $IP -sV | grep "Host:" | cut -d " " -f3,4
```

The `Service Info: Host: <name>; OS: <os>` line is a freebie — record it. Versioned banners (`OpenSSH 7.6p1 Ubuntu 4ubuntu0.3`) usually disclose the Linux distro.

> **Output checkpoint:** per-port `service + version`, OS guess, hostname. Now you can `searchsploit` each version.

---

## Phase 4 — NSE Deepening + Manual Banner Grab

**Goal:** pull more out of the interesting ports — vuln matches, hidden web paths, full banners.

| Trigger / Precondition | Do this |
|---|---|
| Want quick free wins | `sudo nmap $IP -sC` (default-category scripts) |
| Port 80 / web app | `sudo nmap -p80 $IP --script http-enum` → `/robots.txt`, `/wp-admin/`, admin panels |
| Want CVE mapping | `sudo nmap $IP -p 80 -sV --script vuln` (`vulners` prints CVEs + CVSS) |
| Loud-but-fast full picture (NOT if alerts counted) | `sudo nmap $IP -p 80 -A` ( = `-sV -O -sC --traceroute`) |
| Nmap dropped banner detail | manual grab: `nc -nv $IP <PORT>` + `sudo tcpdump -i $IFACE host $MYIP and $IP` in another pane |
| Specific checks | `sudo nmap $IP -p 25 --script banner,smtp-commands` |

```bash
sudo nmap -p80 $IP --script http-enum          # faster than --script discovery
sudo nmap $IP -p 80 -sV --script vuln
nc -nv $IP 31337                               # unusual port → banner-grab first; flags often live here
```

Script locations: `/usr/share/nmap/scripts/*.nse` — `ls /usr/share/nmap/scripts/ | grep <keyword>`. Full docs: https://nmap.org/nsedoc/index.html

> **Output checkpoint:** CVE candidates, discovered web paths, and confirmed real banners (lab flags repeatedly hid in banners on `31337` / `50000` / `version.bind`).

---

## Phase 5 — Save the Results

**Goal:** never lose a slow scan; produce a report-ready artifact.

| Flag | Ext | Use |
|---|---|---|
| `-oN` | `.nmap` | normal (terminal output) |
| `-oG` | `.gnmap` | grepable — one host/line |
| `-oX` | `.xml` | machine-readable, tool ingestion, → HTML |
| `-oA <base>` | all 3 | **default — do this every time** |

```bash
sudo nmap --open -p- $IP -T5 -oX report.xml
xsltproc report.xml -o report.html
firefox report.html
```

> **Output checkpoint:** `report.{nmap,gnmap,xml}` + a clean HTML deliverable. XML is the source of truth — regenerate other formats from it.

---

## Phase 6 — Performance Tuning (Escape Hatch: Scan Too Slow)

**Trigger:** `-p-` / `-sV` taking forever, or `/24` sweep crawling.

| Flag | Controls | Note |
|---|---|---|
| `-T 0..5` | timing template | default `-T3`; `-T4` = fast + accurate baseline |
| `--min-rate <n>` | packets/sec floor | white-box: `1000+` |
| `--max-retries <n>` | retries on no-reply | default 10 → drop to `1` to plough filtered hosts |
| `--initial-rtt-timeout` / `--max-rtt-timeout` | per-probe wait | tune low on fast LAN (`50ms`/`100ms`) |

```bash
sudo nmap -p- --min-rate 1000 -T4 $IP            # fast TCP enum, friendly net
sudo nmap $CIDR -F --min-rate 300 -T4            # fast top-100 /24 sweep
sudo nmap $CIDR -F --max-retries 0               # plough through filtered hosts
```

> **Output checkpoint:** scan finished in time — but speed kills accuracy. **Re-verify any "missing" host/port with a slower second pass** before writing it off.

---

## Phase 7 — Firewall / IDS / IPS Evasion (Escape Hatch: Filtered/Blocked)

**Trigger:** ports mysteriously `filtered`, host "seems down" but should be alive, or alerts are being counted.

| Symptom | Move |
|---|---|
| Need to know *which* ports the firewall rules on | ACK scan: `sudo nmap $IP -p 21,22,25 -sA -Pn -n --disable-arp-ping --packet-trace` (`R`=unfiltered) |
| Ports filtered for no clear reason | **try source port 53 first**: `sudo nmap $IP -p $PORT -sS --source-port 53` (`-g53` = alias) |
| Host "down" but you know it's up | `-Pn` (skip discovery) + `-n` (skip DNS) + `--disable-arp-ping` |
| DNS/UDP service behind tight filter | `sudo nmap -Pn --disable-arp-ping -p53 -sU -sC $IP` (`dns-nsid` auto-fires; manual: `dig CH TXT version.bind @$IP`) |
| Want to bury your IP in logs | decoys: `sudo nmap $IP -p 80 -sS -D RND:5` (or `-D 1.2.3.4,ME,5.6.7.8`) |
| Service exposed only to internal subnet | spoof source: `sudo nmap $IP -n -Pn -p 445 -O -S $SPOOFED_IP -e $IFACE` |
| Corp DNS more permissive than public | `nmap --dns-servers $NS1,$NS2 $TARGET` |

**Critical chaining rule:** once a source-port scan finds the port, your *follow-up* connection must reuse the same source port or the firewall blocks it:

```bash
sudo nmap -g53 --max-retries=1 -Pn -p- --disable-arp-ping $IP
sudo nc -nv -s $MYIP -p53 $IP 50000            # same source port → banner returns
ncat -nv --source-port 53 $IP 50000            # ncat equivalent
```

**Common evasion stack:**

```bash
sudo nmap $IP -p- -sS --source-port 53 -T2 -Pn -n --disable-arp-ping --max-retries 1
```

> **Output checkpoint:** ports that were `filtered` now show `open`; banner grabbed through the same source port.

---

## Decision Tree (Under Exam Pressure)

```
You have:
│
├── just a CIDR / scope
│   └── Phase 1: nmap -sn → live list → Phase 2 on each
│
├── live host, nothing scanned
│   ├── Phase 2: nmap --open -p- -T4   (ALWAYS full range, not top-1000)
│   ├── host "down" but should be up → -Pn -n --disable-arp-ping → Phase 7
│   └── scan crawling → Phase 6 (-T4 --min-rate 1000 --max-retries 1)
│
├── open ports, no versions
│   ├── Phase 3: -sV on those ports (+ -sC for free wins)
│   ├── 139/445 open → -sV → grab NetBIOS hostname
│   └── weird port (31337/50000/8443) → nc -nv banner-grab FIRST
│
├── versions known, need a foothold/vuln
│   ├── Phase 4: --script vuln (vulners → CVEs) ; web → --script http-enum
│   ├── banner looks truncated → nc + tcpdump in 2nd pane
│   └── tcpwrapped / unknown service → manual nc / openssl s_client
│
├── ports show "filtered"
│   ├── --source-port 53 (-g53)  ← try this FIRST, works absurdly often
│   ├── -sA to map which ports the firewall rules on
│   ├── ICMP error vs silence → --reason (rejected vs dropped)
│   └── still nothing → decoys / -S spoof / UDP (-sU -sC on 53)
│
├── alerts being counted (IDS lab / exam)
│   ├── NO -O, NO -A, NO --script vuln
│   ├── sudo nmap -sV --top-ports 10 --disable-arp-ping $IP
│   └── read distro out of SSH/Apache banner (free OS, no -O)
│
└── STUCK > 10 min
    ├── grep ../ATTACK-PATHS.md for the current symptom
    ├── Did you actually run -p-? top-1000 lies. Re-run full range.
    ├── Did you -sV? SERVICE column is a guess. Banner-grab manually.
    ├── "down"/"filtered" → -Pn -n + --source-port 53 + slower -T2
    └── re-verify "missing" port with a slow second pass (speed killed it)
```

---

## Signal → Counter-Move Reference

| You see / observe | Likely cause | Counter-move |
|---|---|---|
| Host marked **down** with `-sn` | alive but blocking ICMP | TCP-SYN ping `-PS22,80,443`, or `-Pn -n --disable-arp-ping` |
| You asked `-PE` but trace shows ARP | same L2 segment — Nmap ARP-pings first | `--disable-arp-ping` to actually test ICMP |
| Host discovery unreliable / fewer hosts | forgot `sudo` — probes silently downgraded | re-run with `sudo` |
| Port `filtered`, no response at all | firewall **dropping** (10 retries → slow) | `--source-port 53`; or `--max-retries 1` to speed the confirm |
| Port `filtered`, fast reply | ICMP type3/code3 **reject** rule | `--reason` confirms; try `--source-port 53` / `-sA` map |
| `SERVICE` says `ibm-db2` but it's FTP | column is a guess from `nmap-services` | `nc -nv $IP $PORT` — trust the real banner, not the column |
| Service shows `tcpwrapped` / `unknown` | custom service, no matching probe | manual `nc` / `openssl s_client`; banner-grab by hand |
| `-sV -p-` taking hours | `-sV` is much slower than SYN | scope `-sV` to known-open ports only; `--stats-every=5s` |
| Lots of hosts "went offline" mid fast scan | `-T5` / low RTT dropped packets | they're alive — re-verify with `-T4` second pass |
| `--top-ports` / `-F` missed the service | static frequency DB, no `31337`/`50000` | always run `-p-` once on a fresh target |
| `nc` connection refused after source-port scan | follow-up didn't reuse source port 53 | `nc -s $MYIP -p53 $IP $PORT` (needs `sudo`) |
| `-sU` confirms open but no banner | UDP needs the app to answer the right probe | add `-sC` (e.g. `dns-nsid` on 53); or `dig CH TXT version.bind @$IP` |
| Status page alert counter spiking | ran `-O` / `-A` / `--script vuln` | stop — `-sV --top-ports 10 --disable-arp-ping`; OS from banner |
| `-S` spoof gets no replies | upstream BCP38 anti-spoofing | expected — only works if you control the path / can sniff |
| Output file vanished | saved to CWD, you `cd`'d away | use full path with `-oA`; timestamp long engagements |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# === 1: Cold start, only a CIDR ===
sudo nmap $CIDR -sn -oA tnet | grep for | cut -d" " -f5

# === 2: Live host → everything ===
sudo nmap --open -p- $IP -T4 -oA full
sudo nmap --open -p- $IP -T4 | grep "/tcp" | wc -l        # count open ports

# === 3: Versions + free wins on open ports ===
sudo nmap $IP -p <OPEN_PORTS> -sV -sC --reason -oA sv

# === 4: SMB hostname leak ===
sudo nmap -p445 $IP -sV | grep "Host:" | cut -d " " -f3,4

# === 5: Web deep enum ===
sudo nmap -p80 $IP --script http-enum
sudo nmap $IP -p 80 -sV --script vuln                     # CVEs via vulners

# === 6: Manual banner grab (Nmap dropped detail) ===
nc -nv $IP 31337
sudo tcpdump -i $IFACE host $MYIP and $IP                 # 2nd pane

# === 7: Report ===
sudo nmap --open -p- $IP -T4 -oX report.xml
xsltproc report.xml -o report.html && firefox report.html

# === 8: Scan too slow ===
sudo nmap -p- --min-rate 1000 -T4 --max-retries 1 $IP

# === 9: "Host seems down" / filtered ===
sudo nmap $IP -p- -Pn -n --disable-arp-ping

# === 10: Filtered ports → source-port evasion (try FIRST) ===
sudo nmap -g53 --max-retries=1 -Pn -p- --disable-arp-ping $IP
sudo nc -nv -s $MYIP -p53 $IP 50000

# === 11: Firewall rule mapping ===
sudo nmap $IP -p 21,22,25 -sA -Pn -n --disable-arp-ping --packet-trace

# === 12: DNS/UDP behind tight filter ===
sudo nmap -Pn --disable-arp-ping -p53 -sU -sC $IP
dig CH TXT version.bind @$IP                              # manual equivalent

# === 13: Alerts counted — quiet OS fingerprint ===
sudo nmap -sV --top-ports 10 --disable-arp-ping $IP

# === 14: Decoys / source spoof ===
sudo nmap $IP -p 80 -sS -D RND:5
sudo nmap $IP -n -Pn -p 445 -O -S $SPOOFED_IP -e $IFACE
```

---

## Quick Reference — Tools / Flags by Function

| Need | Use |
|---|---|
| Live host discovery | `nmap -sn` (+ `--reason`, `--packet-trace`) |
| Skip discovery (it lies) | `-Pn -n --disable-arp-ping` |
| All TCP ports | `-p-` (never just top-1000) |
| Fast triage | `-F` / `--top-ports=N` |
| Stealth scan | `-sS` (default root) |
| Most accurate scan | `-sT` (loud, logs) |
| Firewall mapping | `-sA` (filtered vs unfiltered) |
| UDP service | `-sU` (+ `-sC` for banner) |
| Service + version | `-sV` |
| Default scripts (free wins) | `-sC` |
| Vuln/CVE matching | `--script vuln` (`vulners`) |
| Web path discovery | `--script http-enum` |
| Everything, loud | `-A` (= `-sV -O -sC --traceroute`) |
| Save output | `-oA <base>` (always) |
| XML → HTML report | `xsltproc report.xml -o report.html` |
| Speed | `-T4 --min-rate <n> --max-retries 1` |
| Source-port evasion | `--source-port 53` / `-g53` |
| Manual banner grab | `nc -nv $IP $PORT` (+ `tcpdump`) |
| Source-port nc | `nc -s $MYIP -p53 $IP $PORT` / `ncat --source-port 53` |
| DNS fingerprint | `dig CH TXT version.bind @$IP` |

---

## Top Gotchas (Things That Will Burn You)

1. **Top-1000 lies.** Run `-p-` once on every fresh target — the lab's real services hid on `31337` and `50000`. Skipping the full scan is the #1 nmap mistake.
2. **`SERVICE` column ≠ confirmed service.** It's a static guess from `nmap-services` (it said `ibm-db2` for a ProFTPd port). Always confirm with `-sV` or a manual `nc` banner grab.
3. **`closed`/`filtered` may be a timeout, not reality.** A port marked closed because Nmap timed out is *not* closed. Re-verify with a slower pass / `-Pn`.
4. **On the same L2 segment Nmap ARP-pings even when you ask for `-PE`.** Add `--disable-arp-ping` to actually test ICMP filtering.
5. **Forgetting `sudo`** silently downgrades probes (`-sS`→`-sT`) and wrecks host discovery reliability.
6. **`-sV -p-` is brutally slow.** Don't combine unless patient — scope `-sV` to the ports `-p-` already found.
7. **`-T5` / aggressive RTT on a flaky link drops packets** → real ports/hosts vanish. Use `-T4`; re-verify "missing" with a slow second pass. `--max-retries 0` makes filtered ports invisible.
8. **Default `--max-retries 10`** means `-p-` against filtered hosts can take *hours* (10 unanswered packets/port). Drop to `1` if you trust the network.
9. **`-O` / `-A` / `--script vuln` burn IDS alerts.** When a status page counts alerts, the distro is already in the SSH/Apache banner — `-sV --top-ports 10` gets the OS for free; `-O` just spends points for the same answer.
10. **`-A` invokes `-O`**, which needs root + at least one open *and* one closed port to fingerprint reliably.
11. **Source-port evasion is sticky.** After `-g53` finds the port, the follow-up `nc`/`curl` must *also* use source port 53 or the firewall blocks it. `nc -s <ip> -p53` needs `sudo` (privileged port; may already be bound by a local resolver).
12. **`-S` spoofed source IPs are filtered by upstream BCP38** — you won't get replies unless you control the path. Decoys also multiply packets → slower.
13. **`--source-port` does not change your source IP** — the firewall still sees your real address; only the port is masqueraded.
14. **UDP without `-sC` only proves the port is open** — you need scripts (e.g. `dns-nsid`) or `dig` to extract the banner. UDP scans are slow even for one port.
15. **Without `-Pn`, tight filters make Nmap try TCP discovery first → "host seems down"** and you never scan it. `-Pn -n --disable-arp-ping` on filtered targets.
16. **`-oA` writes to the current dir** — `cd` afterwards and it's gone; re-running the same basename overwrites silently. Use full paths / timestamps.
17. **`--script vuln` includes intrusive checks** and some scripts trip IDS — don't fire at production without authorization.
18. **A banner can lie** (`Apache (Ubuntu)` could be a spoofed nginx) — confirm OS/service with multiple signals.

---

## Related Vault Notes

- `[[01-enumeration]]` — enumeration mindset; tools ≠ knowledge
- `[[02-introduction-to-nmap]]` — syntax skeleton, scan techniques, `-sS` behaviour
- `[[03-host-discovery]]` — `-sn`, ARP vs ICMP, TTL→OS
- `[[04-host-and-port-scanning]]` — six port states, SYN/Connect/ACK, UDP, `-sV`
- `[[05-saving-the-results]]` — `-oA`, XML→HTML with `xsltproc`
- `[[06-service-enumeration]]` — `-sV`, manual `nc`+`tcpdump` banner grab
- `[[07-nmap-scripting-engine]]` — NSE categories, `-A`, `--script vuln`, `http-enum`
- `[[08-performance]]` — timing templates, `--min-rate`, `--max-retries`, RTT
- `[[09-firewall-ids-ips-evasion]]` — `-sA`, decoys, `-S` spoof, `--source-port 53`, DNS proxy
- `[[10-evasion-easy-lab]]` — quiet OS fingerprint via `-sV --top-ports 10`
- `[[11-evasion-medium-lab]]` — UDP/53 `dns-nsid` / `version.bind`
- `[[12-evasion-hard-lab]]` — `-g53` full scan + matching-source-port `nc` banner grab

Triage by symptom: [[../ATTACK-PATHS]]
