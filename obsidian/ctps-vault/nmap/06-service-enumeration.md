# NOTE — Service Enumeration

## ID
6

## Module
Nmap

## Kind
notes

## Title
Section 6 — Service Enumeration

## Description
Use `-sV` to identify exact service names and versions per port, watch packet traces with `--packet-trace`, and grab full banners manually with `nc` + `tcpdump` when Nmap omits info.

## Tags
nmap, service-detection, version-detection, banner-grabbing, netcat, tcpdump

## Commands
- `sudo nmap <IP> -p- -sV`
- `sudo nmap <IP> -p- -sV --stats-every=5s`
- `sudo nmap <IP> -p- -sV -v`
- `sudo nmap <IP> -p- -sV -Pn -n --disable-arp-ping --packet-trace`
- `nc -nv <IP> 25`
- `sudo tcpdump -i <iface> host <my_ip> and <target_ip>`

## Why Service Enum Matters
The point of `-sV` is to map each open port to an *exact* service + version so you can:
- Search exploit databases for that version.
- Cross-reference CVEs.
- Identify the underlying OS via service banners (e.g. `OpenSSH 7.6p1 Ubuntu 4ubuntu0.3` → Ubuntu).

## Workflow
1. **Quick scan first** to see what's open without flooding the network.
2. **Run `-p- -sV` in the background** — this can take minutes to hours.
3. **While it runs, work on what you already know.** Press `[Space]` during a scan to see live progress, or use `--stats-every=5s` for periodic output.
4. **Add `-v` / `-vv`** to print discovered ports as they're found rather than waiting for the final summary.

## Reading the Output
```
PORT      STATE    SERVICE      VERSION
22/tcp    open     ssh          OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
25/tcp    open     smtp         Postfix smtpd
80/tcp    open     http         Apache httpd 2.4.29 ((Ubuntu))
139/tcp   filtered netbios-ssn
445/tcp   filtered microsoft-ds
Service Info: Host: inlane; OS: Linux; CPE: cpe:/o:linux:linux_kernel
```
- **`Service Info`** line gives you OS + hostname for free.
- Versioned banners (`OpenSSH 7.6p1 Ubuntu 4ubuntu0.3`) usually disclose the Linux distro.

## When Nmap Misses Banner Detail
Sometimes the banner has more info than Nmap displays. Confirm by manually banner-grabbing with `nc`:
```bash
nc -nv <IP> 25
# 220 inlane ESMTP Postfix (Ubuntu)
```
Run `tcpdump` in another tmux pane to capture the raw exchange:
```bash
sudo tcpdump -i eth0 host <my_ip> and <target_ip>
```
You'll see the 3-way handshake then a PSH-ACK packet from the server containing the full banner — including the Ubuntu marker that Nmap silently dropped.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Flag inside one of the services | **`HTB{pr0F7pDv3r510nb4nn3r}`** | `nc -nv <IP> 31337` returned `220 HTB{pr0F7pDv3r510nb4nn3r}` — the port (Elite) was speaking ProFTPD with the flag in the banner |

### Reference command
```bash
nc -nv <IP> 31337
# (UNKNOWN) [<IP>] 31337 (?) open
# 220 HTB{pr0F7pDv3r510nb4nn3r}
```

## Key Takeaways
- `-sV` is the difference between guessing and knowing. Run it on every "interesting" port.
- The `Service Info:` line is a freebie — record it.
- Manual `nc` banner grabs catch info Nmap drops.
- Unusual ports (31337, 50000) almost always run something interesting — banner-grab them first.

## Gotchas
- `-sV` is *much* slower than a SYN scan. Don't combine `-sV` with `-p-` unless you're patient.
- Some services (HTTP, RDP) need a probe to respond. Nmap's `nmap-service-probes` file controls this — custom services may show as `tcpwrapped` or just `unknown`.
- A banner can lie. Apache showing `Apache/2.4.29 (Ubuntu)` could be a banner-spoofed nginx — confirm with multiple signals.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|nmap]]
← [[05-saving-the-results]] | [[07-nmap-scripting-engine]] →
<!-- AUTO-LINKS-END -->
