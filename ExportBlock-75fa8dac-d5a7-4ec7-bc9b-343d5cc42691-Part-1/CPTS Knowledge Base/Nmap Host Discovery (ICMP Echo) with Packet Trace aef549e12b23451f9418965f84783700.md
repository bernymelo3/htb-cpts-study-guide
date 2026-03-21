# Nmap Host Discovery (ICMP Echo) with Packet Trace

Category: Enumeration
Command: sudo nmap 10.129.2.18 -sn -oA host -PE --packet-trace --disable-arp-ping
Description: Host discovery (ping scan) using ICMP Echo requests only, with packet trace output enabled.

Notes:
- -sn: host discovery only (no port scan)
- -PE: ICMP echo request
- --disable-arp-ping: do not use ARP discovery on local networks
- --packet-trace: show sent/received packets
- -oA host: output in all formats with base name "host"• Linux / Unix: 64
• Windows: 128
• Network devices: 255
Tags: enumeration, nmap