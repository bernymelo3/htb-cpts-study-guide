# THEORY — Detection & Prevention of Pivoting and Tunneling

## ID
614

## Module
Pivoting, Tunneling, and Port Forwarding

## Kind
notes

## Title
Section 17 — Detection & Prevention

## Description
Defender‑focused overview of how to detect and prevent pivoting, tunneling, and lateral movement — covering baselines, people/processes/technology, MITRE ATT&CK mappings, and practical mitigations.

## Tags
detection, prevention, mitre, defender, blue-team, theory

## TL;DR — What's Important
- **Baseline everything** – know what hosts, NICs, open ports, and protocols are normal. Dual‑homed hosts are a red flag.
- **People are the weakest link** – BYOD and poor user habits can bring attackers inside. Use MFA and security awareness.
- **Network segmentation + firewalls** make pivoting harder. Separate management, production, and user networks.
- **Monitor for non‑standard ports and protocol tunneling** – HTTP on port 444, DNS tunneling, SSH over HTTPS – these are suspicious.
- **Living off the land (LOTL)** is hard to detect – use EDR, SIEM, and behavioral baselines to spot abnormal command shells and netstat changes.

## Concept Overview
Detection and prevention are two sides of the same coin. A well‑defended network starts with a **baseline** – a documented inventory of hosts, applications, network diagrams, and allowed traffic patterns. From there, defenders enforce policies (processes), train users (people), and deploy technical controls (technology). This section maps common offensive TTPs (from this module) to MITRE ATT&CK and suggests specific mitigations: MFA for remote services, firewall rules for non‑standard ports, SIEM correlation to spot tunneling, and out‑of‑band management for network devices.

## Key Concepts

### Baseline & Documentation
What to track:
- DNS records, DHCP scopes, network device backups
- Application inventory and approved software catalog
- List of all enterprise hosts (physical and virtual) with locations
- Users with elevated permissions
- **Dual‑homed hosts** – systems with two or more active NICs
- Up‑to‑date network diagram (use diagrams.net / Draw.io)

### People, Processes, Technology

| Category | Key Actions |
|----------|--------------|
| **People** | Enforce MFA, security awareness training, manage BYOD risks, have an incident response plan |
| **Processes** | Access control policies, host provisioning/decommissioning, change management, periodic audits |
| **Technology** | Firewalls (next‑gen), IDS/IPS, SIEM, EDR, network segmentation, out‑of‑band management |

### MITRE ATT&CK Mappings & Mitigations

| TTP | MITRE Tag | Prevention / Detection |
|-----|-----------|------------------------|
| External Remote Services | T1133 | Firewall rules, VPN only, block internal protocols from reaching the Internet |
| Remote Services (RDP, SSH) | T1021 | MFA, limit remote access users, host‑based firewalls, OOB management for infrastructure |
| Use of Non‑Standard Ports | T1571 | Baseline normal ports; IDS/IPS to flag HTTP on port 444, SSH on port 443, etc. |
| Protocol Tunneling | T1572 | Lock down outbound DNS (only to internal DNS), monitor for beaconing patterns, block unnecessary egress |
| Proxy Use | T1090 | Maintain allow/block lists for IPs/domains; no direct connection to attacker infrastructure |
| Living off the Land (LOTL) | (none) | EDR, command line logging, SIEM correlation, user behavior analytics |

## Why It Matters
As penetration testers, we must provide actionable recommendations to our clients. Understanding how to detect and prevent the techniques we just used (SSH tunnels, SOCKS proxies, netsh portproxy, etc.) turns a “break‑in” into a “fix‑it”. Defenders who implement these controls force attackers to work harder, increasing the chance of detection. This section gives you the blue‑team vocabulary and strategies to close the loop.

## Defender Perspective

### Detection Signs
- **Unusual netstat output** – a Windows host with a `portproxy` rule or a Linux host with a `sshd` listening on a high port.
- **ICMP‑only traffic** – a host that only sends and receives ping but carries a TCP handshake inside (ptunnel‑ng).
- **Dual‑homed hosts** – any host with two active NICs should be investigated; it may be an intentional or accidental bridge.
- **Beaconing** – regular outbound connections to the same external IP on a non‑standard port (C2 over tunneling).
- **SSH from workstations** – SSH clients on user endpoints are unusual unless explicitly approved.

### Mitigations
- **Network segmentation** – place public‑facing services in a DMZ; isolate user workstations from management networks.
- **Out‑of‑band (OOB) management** – infrastructure devices (switches, routers) should only be reachable via a dedicated OOB network.
- **Disable unnecessary protocols** – block SSH/RDP egress from workstations unless required.
- **Application whitelisting** – prevent execution of `socat`, `plink.exe`, `ptunnel-ng` unless approved.
- **Host‑based firewall rules** – limit which ports can accept incoming connections (prevents netsh portproxy from being useful).
- **SIEM correlation** – alert on `portproxy` additions, new listening ports, or SSH tunnels on non‑standard ports.

## Key Takeaways
- **Dual‑homed hosts are the #1 pivot enabler** – document and minimize them. If they must exist, monitor them strictly.
- **MFA stops most lateral movement** – even if an attacker steals a password or hash, they still need the second factor.
- **Baseline is your best friend** – you can't spot “abnormal” if you don't know what “normal” looks like.
- **Defense in depth** – firewall + IDS/IPS + EDR + SIEM + user training. No single control stops all pivoting.
- **LOTL is hard to detect** – focus on behavior (e.g., `netstat` showing a new listening port, `ssh` spawning from a non‑admin user) rather than tool names.

## Gotchas
- **Over‑restrictive policies** can break business functions. Always balance security with usability.
- **MFA fatigue** – attackers can spam MFA push notifications. Use number matching or hardware tokens.
- **BYOD** – you cannot fully secure a device you don't own. Stronger network segmentation and conditional access are needed.
- **False positives** – legitimate admin tools (like `netsh portproxy` for HAProxy) may trigger alerts. Whitelist known good uses.
- **Encrypted tunnels** – HTTPS, SSH, and DNS over TLS hide the payload. You may only detect the tunnel's existence, not its content. Look for anomalous volumes or beaconing patterns.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|pivoting-tunneling]]  
← [[13-ptunnel-icmp]] | [[skills-assessment]] →
<!-- AUTO-LINKS-END -->
