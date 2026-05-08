## ID
610

## Module
File Upload Attacks

## Kind
notes

## Title
Section 1 — Intro to File Upload Attacks

## Description
Explains the risk of insecure file upload features, the types of attacks that arise from weak validation, and why file upload vulnerabilities remain a top web application threat.

## Tags
theory, concept, file-upload, web-attacks, vulnerability

## TL;DR — What's Important
- **Unchecked file uploads lead to RCE:** The most critical outcome is remote command execution via web shells or reverse shells.
- **Validation is everything:** Weak or missing file type verification is the root cause; bypass techniques exploit flawed filters.
- **Not just RCE:** Even restricted upload types can enable XSS, XXE, DoS, or overwrite critical system files.
- **Attack surface is large:** Unauthenticated arbitrary file upload is the worst-case scenario and nearly always leads to server compromise.
- **Defenders must layer controls:** Combine extension whitelisting, content inspection, size limits, virus scanning, and proper file storage permissions.

## Concept Overview
File upload functionality is ubiquitous—social media photos, document sharing, profile pictures. However, every uploaded file lands on the server, and if the application does not properly validate and restrict what can be stored, attackers can deposit malicious scripts. The missing link between a user’s file and code execution is often just an unvalidated extension or content type. This section sets the foundation for understanding how file upload attacks work and why they are consistently rated High/Critical in public vulnerability reports.

## Key Concepts

### Types of File Upload Attacks
- **Arbitrary File Upload (Unauthenticated):** No restrictions on who can upload or what file type—attackers can immediately place a web shell and execute commands.
- **Limited File Upload:** Only certain extensions are accepted, but bypasses exist (e.g., double extensions, mime-type manipulation, magic bytes) to still achieve code execution.
- **Other Injection Attacks:** Malicious files can introduce stored XSS if displayed inline, XXE if XML parsing is performed server-side, or cause DoS via decompression bombs.
- **File Overwrite:** If the upload path is predictable, attackers may overwrite existing scripts, templates, or configuration files.

### Definitions
- **Web Shell** — A script (e.g., PHP, ASPX) placed on a web server that accepts commands from the attacker via HTTP, enabling interactive system access.
- **Reverse Shell** — A script that connects back to an attacker-controlled listener, providing a command-line session without relying on the web server’s document root.
- **File Validation** — The process of checking the extension, MIME type, magic bytes, and/or content of an uploaded file to decide whether it is permitted.

## Why It Matters
File upload flaws appear in countless real-world engagements—from image resizers to document converters. Because they directly allow an attacker to place executable code on a target, they are often the fastest path to full server compromise. Understanding the range of possible attacks (not only RCE) helps testers recognise the risk even when a simple web shell is blocked. For defenders, it’s critical to see file uploads as a multi-layered risk, not just a file extension problem.

## Defender Perspective
- **Detection:** Monitor for unexpected file extensions, unusual POST requests to upload endpoints, and responses that reveal the uploaded file’s URL. WAF rules can block patterns like `<?php`, `<script>`, or ELF headers.
- **Mitigations:**
  - Write an allowlist of safe extensions (e.g., `.jpg`, `.png`, `.pdf`) and enforce it on the server side, not just client-side JavaScript.
  - Check the file’s content type and magic bytes, not only the extension.
  - Store uploaded files outside the webroot; rename them with random identifiers; serve them with a handler that forces download, never direct execution.
  - Apply least privilege: the upload directory should not allow script execution, and the application should run under a restricted account.
- **MITRE ATT&CK:** Initial Access (T1190 Exploit Public-Facing Application) often includes file upload bugs. Persistence and Execution (T1505.003 Web Shell) directly rely on uploaded web shells.

## Key Takeaways
- File upload attacks are not just about hacking; they’re a design-level failure where the server trusts user-provided data more than it should.
- Even “safe” file repositories can be dangerous if the application uses uploaded files in unsafe ways (e.g., including them in PDFs, processing XML, or displaying them as HTML).
- An attacker’s creativity matters: a `.phar` file can compromise PHP, a `.htaccess` can enable code execution, and a `.svg` can carry XSS payloads.
- The best defence is to assume every upload is malicious and design the storage and serving architecture accordingly.

## Gotchas
- Developers often validate only the extension on the client side, which is trivially bypassed with a proxy like Burp.
- Content-Type headers can be spoofed; always check file signatures (magic bytes).
- In shared hosting environments, an uploaded web shell in one vhost can compromise all sites on the same server.