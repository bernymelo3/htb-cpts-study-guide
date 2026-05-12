# NOTE — Laudanum, One Webshell to Rule Them All

## ID
410

## Module
Shells & Payloads

## Kind
notes

## Title
Section 13 — Laudanum, One Webshell to Rule Them All

## Description
Laudanum is a pre-built collection of ready-to-deploy web shells across many languages (asp, aspx, jsp, php, etc.), shipped with Parrot/Kali. Walkthrough uses the `aspx` shell against an IIS app.

## Tags
laudanum, web-shells, aspx, asp, jsp, php, ip-allowlist, ascii-comments

## Commands
- `cp /usr/share/laudanum/aspx/shell.aspx ./demo.aspx` — make local editable copy
- (Edit line 59 of `demo.aspx` — add attacker IP to `allowedIps`)
- (Upload via target's upload form → navigate to `/files/demo.aspx`)
- `systeminfo` — example command issued through the shell form

## What This Section Covers
Laudanum is a multi-language shell repo bundled into Parrot/Kali at `/usr/share/laudanum/`. Each language subdir has a ready shell template — usually just needs the attacker IP injected before upload. Best used against apps that accept file uploads but don't validate content.

## Where Laudanum Lives
```
/usr/share/laudanum/
├── aspx/shell.aspx
├── jsp/shell.jsp
├── php/shell.php
├── asp/...
└── ...
```
On other distros, clone from GitHub.

## Pre-Use Modifications
1. **Copy** the language-appropriate shell to your workspace.
2. **Edit the IP allowlist** — line ~59 of `shell.aspx` has `allowedIps = new string[] { ... }`. Add the attacker IP.
3. **Strip ASCII art / comments** — public payloads ship with author signatures that are AV-signatured.

## Walkthrough — ASPX Shell on IIS Target
Target: `status.inlanefreight.local` (add to `/etc/hosts`).
1. Copy + edit shell file (add IP, optionally strip art).
2. Use the app's "Import Configuration File" upload form to land the file.
3. Note the saved path from the success message (e.g. `\files\demo.aspx`).
4. Browse to `status.inlanefreight.local//files/demo.aspx`.
5. Browser shows the Laudanum form with a `cmd /c` input field.
6. Submit commands (e.g. `systeminfo`); STDOUT appears in the response page.

### Path-format quirk
The app reports `\files\demo.aspx`. In the browser, request `/files/demo.aspx` — Firefox/Chrome will canonicalize `//` to `/`.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Establish a web shell and submit the path of the directory you land in | **C:\Windows\system32\inetsrv** (typical IIS landing dir) | Upload shell, browse to `/files/shell.aspx`, run `cd` or look at the prompt context. The exact path depends on the running IIS worker; submit what `cd` returns from the form. |
| Q2 — Path to Laudanum aspx shell on Pwnbox | **/usr/share/laudanum/aspx/shell.aspx** | `ls /usr/share/laudanum/aspx/`. |

## Key Takeaways
- Laudanum is pre-installed; no clone needed on Pwnbox/Parrot/Kali.
- IP allowlist must match attacker's actual source IP (VPN tunnel, not host) — wrong IP = 403/blank page.
- ASCII art / comments are AV-signatured — strip before upload.
- Use `cd` from the shell form to confirm landing directory.

## Gotchas
- The path returned by the app may use backslashes (`\files\...`) — browser request uses forward slashes.
- If you forget the IP allowlist edit, you'll get blocked even with a successful upload.
- Some implementations randomize the uploaded filename — make sure you note exactly where the file went.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|shells-payloads]]
← [[12-introduction-to-web-shells]] | [[14-antak-webshell]] →
<!-- AUTO-LINKS-END -->
