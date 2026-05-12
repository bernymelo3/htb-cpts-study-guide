# NOTE — Antak Webshell

## ID
411

## Module
Shells & Payloads

## Kind
notes

## Title
Section 14 — Antak Webshell

## Description
Antak is an ASP.NET web shell from the Nishang project — themed like a PowerShell console, with built-in upload/download/encode-and-execute. Each command runs as a new process, ideal for Windows IIS targets.

## Tags
antak, nishang, aspx, powershell, web-shells, ippsec, asp-net

## Commands
- `cp /usr/share/nishang/Antak-WebShell/antak.aspx ./Upload.aspx` — local editable copy
- (Edit line 14 — set credentials: `Disclaimer:ForLegitUseOnly` by default)
- (Upload to target's file upload form)
- Navigate to `/files/Upload.aspx` → login → run PowerShell commands
- `whoami` / `help` / `clear` — sample commands inside the shell

## What This Section Covers
Antak ships with Nishang, the offensive PowerShell toolkit. Unlike Laudanum's basic command runner, Antak feels like a PowerShell prompt in the browser — supports script upload, encoded execution, parses `web.config`, and runs SQL queries. Best against any IIS box.

## ASPX Background
**Active Server Page Extended (ASPX)** = Microsoft's ASP.NET web form format. Server runs the .aspx code, returns HTML. A malicious ASPX shell turns that "server-side code execution" into a backdoor controlled by browser POSTs.

## Antak Capabilities
- PowerShell-style console UI in browser
- Each command runs as a new process
- Execute scripts in memory
- Encode commands (helpful for AV evasion)
- File upload / download
- Parse `web.config`
- Execute SQL queries

## Where Antak Lives
```
/usr/share/nishang/Antak-WebShell/antak.aspx
```

## Pre-Use Modifications
1. **Copy** to workspace.
2. **Edit credentials** — line 14:
   ```csharp
   if (username == "Disclaimer" && password == "ForLegitUseOnly")
   ```
   Change to your own credentials to keep the shell private.
3. **Strip ASCII art / comments** — signature reduction.

## Walkthrough — Antak on IIS Target
1. Add `<target-ip> status.inlanefreight.local` to `/etc/hosts`.
2. Copy + edit Antak `.aspx`.
3. Upload via the app's import/upload form.
4. Navigate to `http://status.inlanefreight.local/files/Upload.aspx`.
5. Login with the credentials you set.
6. Issue PowerShell commands — `whoami`, `dir`, etc.

## Learning Resource — ippsec.rocks
For visual learners, [ippsec.rocks](https://ippsec.rocks/) indexes IPPSEC's HTB YouTube walkthroughs by keyword (search `aspx`, `antak`, `web shell`) and timestamps directly into demo segments.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Path to Antak on Pwnbox | **/usr/share/nishang/Antak-WebShell/antak.aspx** | `locate antak` returns this path. |
| Q2 — User the commands run as (Format: `****\****`) | **NT AUTHORITY\IUSR** (typical IIS) — exact value from the lab: depends on app pool config | Upload Antak, login, run `whoami` in the prompt. |

## Key Takeaways
- Antak is the "PowerShell in a browser" web shell — perfect on IIS.
- Default creds (`Disclaimer:ForLegitUseOnly`) should be changed every deploy.
- Each command spawns a new process — state doesn't persist between submissions. Use `Encode and Execute` for multi-line scripts.
- The `IUSR` / `IIS APPPOOL\...` identity is usually what executes; privesc still needed.

## Gotchas
- Default credentials are public — anyone who finds your shell can log in.
- Backslash-format username output: HTB lab answers may want lowercase, may want the exact format including domain — match the format prompt.
- New process per command means no persistent env vars; use `Encode and Execute` for sequences.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|shells-payloads]]
← [[13-laudanum-webshell]] | [[15-php-web-shells]] →
<!-- AUTO-LINKS-END -->
