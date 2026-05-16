## ID
700

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 23 — ColdFusion - Discovery & Enumeration

## Description
Covers how to identify and enumerate ColdFusion installations through port scanning, file extensions, HTTP headers, error messages, and default files.

## Tags
coldfusion, enumeration, discovery, nmap, cfml, adobe

## Commands
- nmap -p- -sC -Pn <TARGET_IP> --open
- nmap -A -Pn <TARGET_IP>

## What This Section Covers
ColdFusion is an Adobe-owned web application platform built on Java, using ColdFusion Markup Language (CFML) for dynamic web apps. This section teaches how to fingerprint ColdFusion installations during a pentest through port scanning, identifying default file extensions (.cfm, .cfc), checking HTTP headers, recognizing error messages, and locating default admin paths.

## Methodology
1. Run a full port scan with `nmap -p- -sC -Pn <TARGET_IP> --open` and look for default ColdFusion ports (80, 443, 8500, 5500, 1935)
2. Browse to discovered ports and look for directory listings containing `CFIDE` and `cfdocs` folders
3. Check for `.cfm` and `.cfc` file extensions in directory listings or spider results
4. Inspect HTTP response headers for `Server: ColdFusion` or `X-Powered-By: ColdFusion`
5. Navigate to `/CFIDE/administrator/index.cfm` to find the admin login page and identify the ColdFusion version

## Key Takeaways
- ColdFusion default ports: 80 (HTTP), 443 (HTTPS), 1935 (RPC), 25 (SMTP), 8500 (SSL), 5500 (Server Monitor)
- Port 8500 is a strong indicator of ColdFusion — browsing to it often reveals `CFIDE` and `cfdocs` directories in the root
- The `/CFIDE/administrator` path loads the admin login page and typically reveals the exact ColdFusion version
- Known CVEs include command injection (CVE-2020-24450), arbitrary file read (CVE-2020-24449), and XSS (CVE-2019-15909)
- ColdFusion error messages are verbose and leak technology details useful for enumeration

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| What ColdFusion protocol runs on port 5500? | Server Monitor | Reading the default ports table in the section material |
