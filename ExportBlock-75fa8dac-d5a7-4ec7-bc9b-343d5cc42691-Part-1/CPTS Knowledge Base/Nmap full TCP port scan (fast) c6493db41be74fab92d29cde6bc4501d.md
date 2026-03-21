# Nmap full TCP port scan (fast)

Category: Enumeration
Command: nmap <IP> -p- --min-rate 1000 -T4
Description: Fast scan of all 65535 TCP ports to quickly identify open services. Good as an initial discovery scan before targeted version/script scans.
Tags: enumeration, nmap