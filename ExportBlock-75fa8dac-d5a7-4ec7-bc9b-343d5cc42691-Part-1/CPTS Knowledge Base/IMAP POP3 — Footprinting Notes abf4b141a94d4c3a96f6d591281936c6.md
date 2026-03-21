# IMAP / POP3 — Footprinting Notes

Category: Enumeration
Command: # Ports
# IMAP: 143 (plain) / 993 (SSL)
# POP3: 110 (plain) / 995 (SSL)

# IMAP (TAG required)
1 LOGIN user pass
2 LIST "" 
3 SELECT INBOX
4 FETCH 1 BODY[]
5 LOGOUT

# POP3
USER robin
PASS robin
LIST
RETR 1
QUIT

# Footprinting (Nmap)
sudo nmap <IP> -sV -p110,143,993,995 -sC

# SSL connections
# IMAP
openssl s_client -connect <IP>:imaps
# POP3
openssl s_client -connect <IP>:pop3s

# cURL (IMAPS)
curl -k 'imaps://<IP>' --user robin:robin -v

# SSL cert recon (from nmap)
# commonName=mail1.inlanefreight.htb
# organizationName=Inlanefreight
# emailAddress=cry0l1t3@inlanefreight.htb

# Lab flow
# 1) SMTP enum → user robin
# 2) Test creds robin:robin
# 3) Login IMAP
# 4) LIST folders → http://DEV.DEPARTMENT.INT
# 5) SELECT + FETCH emails → flag

# Gotchas
# - Always use TAG for IMAP commands
# - INBOX empty ≠ nothing; LIST all folders
# - Nested folder names may use dots
# - " 0 EXISTS" means selected folder empty
Description: IMAP/POP3 email retrieval footprinting: ports, key commands, nmap/SSL cert recon, and lab flow (robin:robin → IMAP folders → flag).
Tags: enumeration, nmap