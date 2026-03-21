# Question 2 +1

Category: Enumeration
Command: sudo nmap 10.129.x.x -sC -sV -p25

telnet 10.129.x.x 25
VRFY root
VRFY admin

smtp-user-enum -M VRFY -U /usr/share/wordlists/... -t 10.129.x.x
Description: SMTP Footprinting lab — banner/hostname discovery + VRFY user enumeration. Check the SMTP EHLO response for the server hostname (e.g., myhostname / mail*.inlanefreight.htb).
Tags: enumeration