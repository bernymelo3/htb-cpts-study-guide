# HTB - DNS Enumeration via NS + Zone Transfer (AXFR)

Category: Enumeration
Command: # NS lookup (when reverse PTR is NXDOMAIN)
dig @10.129.6.37 inlanefreight.htb NS

# alternative
nslookup -type=NS inlanefreight.htb 10.129.6.37

# zone transfer attempt (common next step)
dig @10.129.6.37 inlanefreight.htb AXFR

# if NS is known, try against that host too
# (replace ns1.inlanefreight.htb with your NS result)
dig @10.129.6.37 inlanefreight.htb AXFR
# or
# dig @ns1.inlanefreight.htb inlanefreight.htb AXFR
Description: When reverse DNS returns NXDOMAIN (no PTR record), enumerate the FQDN by asking the target DNS server for the domain’s NS record:
- dig @<dns_ip> <domain> NS
Expected answer looks like: inlanefreight.htb IN NS ns1.inlanefreight.htb.

Next common HTB step: attempt a zone transfer (AXFR) to dump DNS records:
- dig @<dns_ip> <domain> AXFR
(Usually works only if misconfigured.)
Tags: enumeration, nmap