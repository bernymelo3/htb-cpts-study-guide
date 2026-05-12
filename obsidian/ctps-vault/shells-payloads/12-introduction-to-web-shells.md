# THEORY — Introduction to Web Shells

## ID
409

## Module
Shells & Payloads

## Kind
theory

## Title
Section 12 — Introduction to Web Shells

## Description
Web shells are browser-accessible code execution backdoors uploaded to a web server. Common foothold for external pentests because the perimeter is often hardened against SMB/auth attacks but web upload flaws remain.

## Tags
web-shells, file-upload, php, jsp, aspx, foothold, external-pentest

## TL;DR — What's Important
- **Web shell** = browser-based RCE on a web server.
- Reach: needs a file upload or RCE vulnerability + writable webroot.
- Languages match the server stack: PHP, JSP, ASP.NET (`.aspx`), etc.
- Usually a temporary foothold — upgrade to a proper reverse shell ASAP.

## What This Section Covers
External pentests rarely find vulnerable SMB or unsigned auth; clients harden their perimeters. What remains is web apps — file upload bypasses, RFI/LFI, command injection, SQLi to file write. A web shell is the simplest landing zone for those vectors.

## Common Routes to a Web Shell
| Vector | How |
|--------|-----|
| File upload form | Upload `.php`/`.jsp`/`.aspx` (often bypassing client-side filter or Content-Type check) |
| Self-registration with profile picture | Smuggle a web shell as a "picture" upload |
| Tomcat / Axis2 / WebLogic | Native `.WAR` deployment functionality |
| Misconfigured FTP into webroot | Drop the file straight into served content |
| SQLi `INTO OUTFILE` | Write the shell from a SQL injection |
| RFI / unrestricted include | Make the server execute remote code directly |

## Web Shell Workflow (Generic)
1. Find an upload or RCE primitive.
2. Bypass any filter (Content-Type, magic bytes, extension allowlist).
3. Upload the shell to a path you can browse to.
4. Navigate to that path in the browser.
5. Issue commands through the web shell form/URL.
6. Use the shell to upgrade to a proper reverse shell.

## Why Not Stay In The Web Shell?
- Some apps auto-delete uploads after a period.
- Non-interactive — chaining commands (`whoami && hostname`) often fails.
- Web shells leave forensic artifacts (file on disk, web logs).
- Slow for enumeration vs. an interactive session.

## Key Takeaways
- Web shells are a stepping stone, not a destination — pivot to interactive reverse shell fast.
- Language of the shell must match the server stack (PHP shell on a Tomcat box won't execute).
- File upload bypasses are tested in the [[../web-attacks/00-overview|web-attacks]] / file-uploads module; this section assumes you already got the file landed.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|shells-payloads]]
← [[11-spawning-interactive-shells]] | [[13-laudanum-webshell]] →
<!-- AUTO-LINKS-END -->
