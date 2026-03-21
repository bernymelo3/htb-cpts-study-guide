# Guessing the OS withouth noise

Category: Enumeration
Command: curl -I http://10.129.25.6
Sequence 

sudo nmap -O --disable-arp-ping -Pn 10.129.33.182


sudo nmap -sn -PE --packet-trace --disable-arp-ping <redacted-ip>


sudo nmap -sV -p22,80,110,139,143,445,10001 --disable-arp-ping -Pn <redacted-ip>