---
id: gs-103
module: Getting Started
kind: note
title: "Service Scanning & Network Enumeration"
description: "Nmap scanning techniques, banner grabbing, SMB/FTP/SNMP enumeration, port classification, common service defaults."
tags: [nmap, port-scanning, banner-grabbing, smb, ftp, snmp, service-enumeration, os-fingerprinting]
---

# Service Scanning & Network Enumeration

## Nmap Scan Strategies

### Quick Scan (1000 Most Common Ports)
```bash
nmap $TARGET_IP
```
**Output:** List of open/closed/filtered ports.

### Full TCP Port Scan (All 65535 Ports)
```bash
nmap -p- $TARGET_IP
# or for speed:
nmap -p- --open $TARGET_IP  # only show open ports
```

### Service & Version Detection
```bash
nmap -sV $TARGET_IP     # detect service versions
nmap -sC $TARGET_IP     # run default NSE scripts
nmap -sV -sC -p- $TARGET_IP  # comprehensive (slow, ~3-10 min for full scan)
```

### Aggressive Scan (Service + OS Fingerprint + Scripts)
```bash
nmap -A -p- $TARGET_IP  # equals -sV -sC -O (OS detection)
```

### UDP Scan (for SNMP, DNS, DHCP)
```bash
nmap -sU -p 53,161,162 $TARGET_IP  # slow; use if TCP sparse
```

### Scan Specific Ports (After Initial Recon)
```bash
nmap -sV -sC -p 21,22,80,445,3389 $TARGET_IP
```

---

## Common TCP/UDP Ports & Services

| Port(s) | Protocol | Service | Typical Vuln Category |
|---------|----------|---------|----------------------|
| 20/21 | TCP | FTP | Weak auth, anonymous access, plaintext passwords |
| 22 | TCP | SSH | Weak keys, version exploits (rare), key theft |
| 23 | TCP | Telnet | Plaintext, unencrypted (legacy) |
| 25 | TCP | SMTP | Relay abuse, open relay |
| 80 | TCP | HTTP | Web vulns (SQLi, XSS, auth bypass, RCE) |
| 139 | TCP | NetBIOS | SMB enumeration |
| 161/162 | TCP/UDP | SNMP | Community string brute-force, MIB disclosure |
| 389 | TCP/UDP | LDAP | Anonymous bind, enumeration |
| 445 | TCP | SMB | File share access, EternalBlue (old), lateral movement |
| 3306 | TCP | MySQL | Weak auth, unencrypted, public-facing (rare but dangerous) |
| 3389 | TCP | RDP | Weak auth, BlueKeep (old), token impersonation |
| 5432 | TCP | PostgreSQL | Weak auth, public-facing |
| 5900 | TCP | VNC | Weak auth, no encryption option |
| 8080/8443 | TCP | HTTP/HTTPS Alt | Web app (Tomcat, Jenkins, etc.) |
| 9000-9999 | TCP | Various | Dev frameworks, admin panels |

**Exam tip:** Memorize top 20 ports; recognize them instantly to prioritize enum.

---

## Banner Grabbing (Manual Service ID)

**Netcat method:**
```bash
nc -nv $TARGET_IP $PORT
# Waits for service to send banner; Ctrl+C to exit
```

**Curl method (HTTP):**
```bash
curl -I http://$TARGET_IP:$PORT
```

**Socat method:**
```bash
socat - TCP4:$TARGET_IP:$PORT
```

**Example output:**
```
SSH-2.0-OpenSSH_7.2p2 Ubuntu 4ubuntu2.8
Apache/2.4.41 (Ubuntu)
FTP 220 vsFTPd 3.0.3
```

**Why:** Services often identify themselves; version number helps find exploits.

---

## SMB Enumeration (Port 445/139)

### List Shares (Anonymous / Guest)
```bash
smbclient -N -L \\\\$TARGET_IP
# -N = no password, -L = list shares
```

### Connect to Share with Creds
```bash
smbclient -U username \\\\$TARGET_IP\\sharename
# Password prompt; then: ls, cd, get, put commands
```

### Enumerate OS / Workgroup / Hostname
```bash
nmap --script smb-os-discovery.nse -p445 $TARGET_IP
```

### Common shares to check:
- `print$` (printer drivers)
- `users` (user home dirs — often writable or contain sensitive files)
- `shares` (public file share)
- `admin$`, `c$`, `d$` (admin shares, restricted access)

---

## FTP Enumeration (Port 21)

### Anonymous Login Attempt
```bash
ftp $TARGET_IP
# Login: anonymous
# Password: anonymous (or leave blank)
# Commands: ls, cd, get, put, quit
```

**What to look for:**
- Writable directories (upload shell)
- Cleartext files (credentials, configs)
- Symlinks to sensitive system files (if allowed)

---

## SNMP Enumeration (Port 161/UDP)

### Walk OID Tree (Requires Community String)
```bash
snmpwalk -v 2c -c public $TARGET_IP 1.3.6.1.2.1.1.5.0
# -v 2c = SNMP v2c, -c public = community string "public"
```

**Default community strings to try:**
- `public` (read)
- `private` (read+write, rarely exposed)
- `community` (sometimes)

### Brute-Force Community Strings
```bash
onesixtyone -c /usr/share/seclists/SNMP/common-snmp-strings.txt $TARGET_IP
```

**What SNMP reveals:**
- System uptime, hostname, OS version
- Running processes (sometimes with passwords in cmdline args)
- Routing tables, network interfaces
- Installed software versions

---

## OS Fingerprinting

### Via Nmap
```bash
nmap -O $TARGET_IP  # OS detection (requires root/sudo, some open ports)
nmap -A $TARGET_IP  # includes OS detection
```

### Via Service Banners
- **SSH version** → Ubuntu version mappable (e.g., 8.2p1-4ubuntu0.1 = Ubuntu 20.04)
- **Apache version + PHP version** → Common OS combos
- **Samba version** → Usually reveals Linux distro

### Via HTTP Headers
```bash
curl -I http://$TARGET_IP | grep -i server
# e.g., "Apache/2.4.41 (Ubuntu)" → Linux, specifically Ubuntu
```

---

## Common Misconfigurations

| Service | Misconfiguration | Impact |
|---------|------------------|--------|
| FTP | Anonymous access enabled | Download/upload files, including binaries |
| SMB | Null session allowed | Enumerate shares, users, policies without creds |
| SNMP | Default community string | Full system disclosure |
| HTTP | Version banner exposed | Direct CVE search (e.g., Apache 2.4.39 → known RCE) |
| SSH | Root login allowed | Brute-force root account |
| MySQL/PostgreSQL | Public-facing, weak password | Direct database access, data theft |
| RDP | Public, weak creds | Remote desktop takeover, lateral movement |

---

## Scan Output Interpretation

```
PORT     STATE      SERVICE       VERSION
21/tcp   open       ftp           vsftpd 3.0.3
22/tcp   open       ssh           OpenSSH 7.2p2 Ubuntu
80/tcp   open       http          Apache httpd 2.4.41
139/tcp  open       netbios-ssn   Samba smbd 4.6.2
445/tcp  open       microsoft-ds  Samba smbd 4.6.2
3389/tcp filtered   rdp           (no response)
```

**Interpretation:**
- FTP 3.0.3 → searchsploit vsFTPd 3.0.3
- SSH 7.2p2 → old, check for user enum / key vulns
- Apache 2.4.41 → check HTTP methods, mod_status exposure
- Samba 4.6.2 → check for EternalBlue (unlikely, but old)
- RDP filtered → firewall blocks, skip for now

---

## Exam Speed Tips

1. **Run nmap -p- in background** while you enumerate other services manually
2. **Save output:** `nmap -sV -sC -p- $IP > nmap.txt`
3. **For web ports, do web enum immediately** (don't wait for full nmap)
4. **Focus on open ports only** — filtered ports are behind firewall, skip them
5. **Prioritize services by risk:** Web (80/443) > SMB (445) > SSH (22) > FTP (21) > others
