# HTB - SMB Enumeration Box (10.129.202.5)

Category: Enumeration
Command: nmap -sV -sC -p139,445 10.129.202.5

rpcclient -U "" 10.129.202.5
# inside rpcclient:
#   srvinfo
#   querydominfo
#   netshareenumall

# get share path (null session / no password):
rpcclient -U "" 10.129.202.5 -N -c "netsharegetinfo sambashare"

smbclient //10.129.202.5/sambashare
cd contents
get flag.txt
Description: SMB enum workflow:
- Nmap: found Samba smbd 4.6.2 on 139/445
- rpcclient (anon/null session):
  - srvinfo → InlaneFreight SMB server (Samba, Ubuntu)
  - querydominfo → domain DEVOPS
  - netshareenumall → share sambashare (remark: InFreight SMB v3.1)
  - netsharegetinfo sambashare → full path (use -N to avoid password prompt)
- smbclient: access sambashare/contents and retrieve flag.txt
Tags: enumeration, nmap