# Nmap HTTP service + vuln scripts

Category: Enumeration
Command: sudo nmap 10.129.2.28 -p 80 -sV --script vuln
Description: Run version detection on port 80 and execute the "vuln" NSE category scripts to identify common web/service vulnerabilities.
Tags: enumeration, nmap, web