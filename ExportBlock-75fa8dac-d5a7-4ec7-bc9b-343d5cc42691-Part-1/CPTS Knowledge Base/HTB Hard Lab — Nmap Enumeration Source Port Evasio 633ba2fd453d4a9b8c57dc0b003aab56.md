# HTB Hard Lab — Nmap Enumeration: Source Port Evasion (53/DNS)

Category: Enumeration
Description: Moved to page content (markdown).
Tags: enumeration, nmap

### Context

This technique came up in a **Hack The Box hard lab** where a service was effectively “hidden” behind filtering rules.

### Concept (source port technique)

Some firewalls and IDS/IPS rulesets are more permissive with traffic that *appears* to come from a trusted source port. A common example is **DNS (port 53)**, which is frequently allowed so DNS resolution keeps working.

By setting the **source port** of your probe or connection to a trusted value, you may be able to reach services that otherwise look **filtered** or appear unreachable.

### When to try it (signals)

- A port looks **closed or filtered** in a normal scan, but context suggests it should be reachable.
- The lab explicitly mentions **firewall rules** or **IDS/IPS evasion**.
- Hints suggest a **hidden** or unexpected service.
- A normal connection attempt times out, but succeeds when using a trusted source port.

### Why port 53 is common

- DNS traffic is almost universally allowed through firewalls.
- Some IDS/IPS setups whitelist 53 to reduce the risk of breaking DNS.
- Other source ports that are sometimes treated as “trusted” in misconfigured environments include **80**, **443**, and **3306**.

### Workflow summary (HTB hard lab)

1. Run a standard scan and note what is clearly open.
2. If a hidden service is suspected, repeat probing using a trusted source port (often 53).
3. If you cannot bind to the chosen source port locally, stop the conflicting local service temporarily.
4. Re-attempt the connection with the trusted source port and capture any banner or service response for further enumeration.
5. Restore the local service after testing.

### Commands used (lab)

#### Enumeration with source port spoofing

```bash
sudo nmap -sV -Pn --disable-arp-ping --source-port 53 <target-ip>
```

#### Connect to a filtered service with a trusted source port

```bash
sudo ncat -nv --source-port 53 <target-ip> <port>
```

#### If port 53 is already in use (find the process)

```bash
sudo ss -lpn sport = :53
```

#### Stop the conflicting resolver (example)

```bash
sudo kill <PID>
# or
sudo systemctl stop systemd-resolved
```

#### Restore DNS after testing (example)

```bash
sudo systemctl restart dnsmasq
# or
sudo systemctl start systemd-resolved
```

#### Passive version detection (quieter alternatives)

```bash
nc -nv <target-ip> 22
curl -I http://<target-ip>
```

### Beyond port 53: other evasion ideas (lab notes)

#### Key takeaway

The technique is not specifically about port 53. It is about **finding what the firewall trusts** and impersonating that traffic. In practice, you test multiple source ports until one bypasses the filter.

#### Other trusted source ports to try

```bash
sudo ncat -nv --source-port 80 <target-ip> <port>
sudo ncat -nv --source-port 443 <target-ip> <port>
sudo ncat -nv --source-port 3306 <target-ip> <port>
sudo ncat -nv --source-port 21 <target-ip> <port>
```

Common examples:

- 53: DNS
- 80: HTTP
- 443: HTTPS
- 21: FTP
- 3306: MySQL
- 8080: HTTP-Alt

#### ACK scan (firewall rule mapping)

```bash
sudo nmap -sA --source-port 53 -Pn <target-ip>
```

#### Fragmentation (split packets)

```bash
sudo nmap -f -Pn --disable-arp-ping <target-ip>
sudo nmap -ff -Pn --disable-arp-ping <target-ip>   # double fragmentation
```

#### Decoy scanning (hide your real IP)

```bash
sudo nmap -D RND:10 -Pn <target-ip>         # 10 random decoys
sudo nmap -D 192.168.1.1,ME <target-ip>     # specific decoy + your real IP
```

#### Timing evasion (slow scans)

```bash
sudo nmap -T0 -Pn <target-ip>   # paranoid - very slow
sudo nmap -T1 -Pn <target-ip>   # sneaky
```