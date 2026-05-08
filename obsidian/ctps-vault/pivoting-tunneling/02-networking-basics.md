# THEORY — The Networking Behind Pivoting

## ID
601

## Module
Pivoting, Tunneling, and Port Forwarding

## Kind
notes

## Title
Section 2 — The Networking Behind Pivoting

## Description
A simple refresher on IP addresses, NICs, routing tables, and how these concepts help you find pivot opportunities.

## Tags
networking, theory, ip-addressing, routing, pivoting, beginner

## TL;DR — What's Important

- **A computer needs an IP address to be on a network** – no IP = not reachable.

- **NIC = Network Interface Card** – each NIC has its own IP address. More NICs = more networks reachable.

- **Dual‑homed hosts have two or more NICs** – they can talk to different networks. These are your pivot targets.

- **The routing table tells the computer where to send traffic** – if a destination isn't in the table, it goes to the default gateway.

- **Always run `ifconfig` (Linux) or `ipconfig` (Windows) when you land on a host** – look for extra IP addresses that hint at hidden networks.

## Concept Overview

Before you can pivot, you need to understand the basics of how computers talk to each other. Every computer on a network has an IP address (like a phone number) assigned to a Network Interface Card (NIC). A computer can have many NICs – physical (plugged into cables) or virtual (VPN, VMs). Each NIC can connect to a different network.

When you compromise a host, check its IP addresses. If you see two different IP ranges (e.g., one `192.168.1.10` and another `10.10.10.5`), that host can reach two separate networks. That's your pivot opportunity.

The routing table is like a map. It tells the computer: "If you want to reach this network, send the packet out that NIC and maybe through that gateway." Without the right routes, traffic won't go where you want.

## Key Concepts

### NICs and IP Addresses

| Command | OS | What it shows |
|---------|-----|----------------|
| `ifconfig` | Linux / macOS | All NICs, their IPs, and status |
| `ipconfig` | Windows | All adapters, IPs, and default gateway |

**Example from the lesson (Linux):**

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|pivoting-tunneling]]  
← [[01-intro]] | [[05-ssh-port-forwarding]] →
<!-- AUTO-LINKS-END -->
