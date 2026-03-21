# Nmap SMTP enum (banner + smtp-commands)

Category: Enumeration
Command: sudo nmap 10.129.2.28 -p 25 --script banner,smtp-commands
Description: Enumerate SMTP on port 25: grab service banner and list supported SMTP commands via NSE scripts.
Tags: enumeration, nmap