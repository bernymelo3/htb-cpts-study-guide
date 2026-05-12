# THEORY — Detection & Prevention

## ID
414

## Module
Shells & Payloads

## Kind
theory

## Title
Section 17 — Detection & Prevention

## Description
Blue-team view of everything from sections 1–16 — MITRE ATT&CK mapping (Initial Access, Execution, C2), what events to watch for (uploads, anomalous user actions, anomalous netflow), and high-level mitigations (sandboxing, least priv, segmentation, firewalls).

## Tags
detection, defense, blue-team, mitre-attack, c2, netflow, defender, mitigations

## TL;DR — What's Important
- Map every offensive technique to **MITRE ATT&CK** for reporting and detection design.
- Most-exposed events: file uploads, suspicious non-admin commands, anomalous netflow.
- Catch C2 via baselining: top talkers, unique sites, heartbeats, off-hours traffic.
- Mitigations stack — application sandboxing, least privilege, segmentation, L7 firewalls.

## What This Section Covers
Defense view of payload delivery, shell sessions, and C2. Helps the offense understand *what to evade* and gives the report a defensive recommendation chapter that's grounded in standards.

## MITRE ATT&CK — Relevant Tactics
| Tactic | Relevance to this Module |
|--------|--------------------------|
| **Initial Access** | Public-facing app exploitation, file uploads, misconfigured SMB/auth. Bastion-host compromise. |
| **Execution** | Most of the module — payload delivery + execution via PS one-liners, MSF, web shells, uploaded binaries. |
| **Command & Control** | Result of established shell — common ports (80/443), DNS, NTP, Slack/Teams covers, encrypted/obfuscated channels. |

## Events Worth Alerting On
- **File uploads** to web apps — log + AV-scan + WAF rules.
- **Non-admin users running admin commands** — `whoami`, `net user`, `net localgroup`, PowerShell on a marketing user.
- **Anomalous net sessions** — heartbeat on port 4444 (Meterpreter default), bulk GETs/POSTs, NetFlow deltas vs baseline.

## Network Visibility Stack
- Documented topology diagrams (Draw.io, NetBrain).
- Vendor dashboards (Meraki, Ubiquiti, Check Point, Palo Alto) — L7 visibility built in.
- NetFlow + SIEM for top talkers, unique destinations.
- Deep Packet Inspection appliance — catches plain-text shell traffic even on common ports.

### Concrete example
A reverse-shell over plain TCP (e.g. `nc`) is fully decodable in Wireshark — you can read the `net user hacker /add` command verbatim. Encrypted C2 (Meterpreter HTTPS, Cobalt Strike) defeats this — moves the detection to behavioral.

## End Device Protection
- Keep Windows Defender (Real-Time + Firewall) enabled across all profiles.
- Patch management discipline — close MS17-010-class windows fast.
- AV on servers too — perf hit is real but limits payload execution.
- For Linux: file integrity monitoring, auditd, SELinux/AppArmor profiles.

## Mitigations Reference
| Layer | Control |
|-------|---------|
| App | Sandboxing — limit RCE blast radius |
| OS / IAM | Least privilege — non-admin daily accounts; no domain-admin runas |
| Network | Segmentation — DMZ for internet-facing services; STIG-hardened bastions |
| Network | L7 firewalls — allow-list outbound, NAT, deep inspection |
| Endpoint | EDR, AV, Defender, command-line logging, PS ScriptBlock logging |

## Key Takeaways
- The same techniques you use in sections 1–16 generate detectable signals — knowing which makes your tradecraft stealthier.
- Network visibility ≠ packet capture; baselining (NetFlow + top talkers) is where most C2 gets caught.
- Plain-text shells (raw `nc`) are basically self-reporting; modern engagements need encrypted C2 frameworks.
- Defense-in-depth: no single control stops shells/payloads — the stack does.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|shells-payloads]]
← [[16-skills-assessment-live-engagement]] | _end_ →
<!-- AUTO-LINKS-END -->
