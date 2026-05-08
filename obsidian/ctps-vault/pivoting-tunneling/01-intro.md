# THEORY — Introduction to Pivoting, Tunneling, and Port Forwarding

## ID
600

## Module
Pivoting, Tunneling, and Port Forwarding

## Kind
notes

## Title
Section 1 — Introduction to Pivoting, Tunneling, and Port Forwarding

## Description
Learn what pivoting, tunneling, and lateral movement are, how they differ, and why you need them to move through segmented networks.

## Tags
pivoting, tunneling, port-forwarding, lateral-movement, theory, beginner

## TL;DR — What's Important

- **Pivoting = moving DEEPER** – using a hacked host to reach a completely different network segment (like jumping from an office network to a factory control network).

- **Lateral movement = moving WIDER** – hopping between hosts on the same network using stolen passwords or hashes.

- **Tunneling = hiding your traffic** – wrapping attack commands inside normal-looking protocols (HTTP, DNS, SSH) to sneak past firewalls.

- **A single dual‑homed host (two network cards) is pure gold** – it can bridge two worlds and become your pivot point.

- **You will often need all three** – first move laterally to a jump host, then pivot to a new segment, then tunnel your traffic to avoid detection.

## Concept Overview

When you first break into a network, you usually can't reach everything. Companies split their networks into segments (like putting sensitive servers on a separate VLAN). Pivoting lets you use a compromised computer as a stepping stone to access those hidden segments.

Think of it like this: you break into the reception area of a building, but the valuable stuff is in a locked back room. If you find a staff member who has keys to both areas, you can use them to get into the back room. That staff member is your "pivot host".

Tunneling is a special trick inside pivoting. You hide your attack traffic inside something that looks normal, like wrapping a key inside a teddy bear before mailing it.

## Key Concepts

### Simple Comparison Table

| Technique | What it does | Analogy |
|-----------|--------------|---------|
| **Lateral Movement** | Jump to another computer on the SAME network | Moving sideways between desks in the same office |
| **Pivoting** | Jump to a computer on a DIFFERENT network | Using a hallway door to enter a different building wing |
| **Tunneling** | Hide traffic inside another protocol | Sending a secret letter inside a birthday card |

### Other Names for a Pivot Host

- Proxy
- Foothold
- Beachhead
- Jump box / Jump host

### What Makes a Good Pivot Host?

- Has **two or more network interfaces** (dual‑homed)
- Has access to a network segment you can't reach directly
- Allows you to run tools or forward ports

## Why It Matters

Without pivoting, you would only ever attack hosts directly reachable from your attack box. In real pentests, that's rarely enough. The most valuable targets (domain controllers, database servers, SCADA systems) are often hidden behind firewalls or on isolated VLANs.

Pivoting turns one compromised workstation into a gateway to the entire internal network.

## Defender Perspective

### How to Spot Pivoting

- A workstation making connections to many different internal IPs (scanning)
- Unusual port forwarding rules on a host (`netstat -an` shows listening ports that don't match normal services)
- A host that has two active network interfaces (check with `ipconfig` or `ifconfig`)

### How to Stop It

- **Segment carefully** – don't give one host access to two sensitive networks unless absolutely necessary
- **Monitor outbound connections** – look for internal hosts connecting to unexpected internal IPs
- **Disable unnecessary protocols** – block SSH, RDP, or SMB from workstations if they don't need them
- **Use jump hosts with auditing** – if you must have a dual‑homed host, log everything on it

### MITRE ATT&CK References

- **T1090** – Proxy (pivoting)
- **T1572** – Protocol Tunneling
- **T1021** – Remote Services (lateral movement)

## Key Takeaways

- Always check for **extra network interfaces** when you first compromise a host – that's your clue for pivoting.
- Lateral movement uses the **same credentials** to hop between hosts on the same network.
- Pivoting requires a host that **bridges two networks**.
- Tunneling is about **stealth** – it hides what you're really doing.
- You can chain multiple pivots: attack host → pivot 1 → pivot 2 → target.

## Gotchas

- Don't confuse **port forwarding** with pivoting – port forwarding is just one way to pivot.
- Some tools (like nmap) don't work through a pivot unless you use `proxychains` or a similar wrapper.
- Pivoting adds **latency** – your scans will be slower, so be patient.
- If you pivot through a host, all your traffic looks like it comes from that host – be careful not to crash it with too many requests.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|pivoting-tunneling]]  
[[02-networking-basics]] →
<!-- AUTO-LINKS-END -->
