## ID
106

## Module
Attacking Common Services

## Kind
notes

## Title
Section 6 — Latest FTP Vulnerabilities

## Description
CVE-2022-22836 – an authenticated path traversal and arbitrary file write vulnerability in CoreFTP before build 727, exploitable via a crafted HTTP PUT request.

## Tags
theory, vulnerability, ftp, directory-traversal, cve, coreftp

## TL;DR — What's Important
- **CVE-2022-22836** affects CoreFTP < build 727. It allows an authenticated user to write files anywhere on the filesystem using a crafted HTTP PUT request.
- **The attack vector** is not FTP commands but an **HTTP PUT** request to the same service (CoreFTP listens on port 80/443 as well).
- **Directory traversal payload** `../../../../../../whoops` breaks out of the web root, bypassing path restrictions.
- **Privileges matter** – the service writes the file with its own permissions (often SYSTEM or a privileged user).
- **The exploit is simple** – a single `curl` command with `--path-as-is` to prevent normalizing the traversal sequence.

## Concept Overview
CoreFTP is a Windows‑based FTP server that also includes a web interface. Versions before build 727 contain a vulnerability in how they handle HTTP PUT requests. Even though the user is authenticated (basic auth), the application does not properly sanitize the file path. By including directory traversal sequences (`../` or `../../`), an attacker can write arbitrary content to any location on the server’s filesystem. This can lead to remote code execution (e.g., writing a webshell into the web root) or overwriting critical system files.

The vulnerability is mapped to the “Concept of Attacks” pattern:  
**Source** = the crafted PUT request with traversal path and content.  
**Process** = the HTTP request handler that fails to validate the path.  
**Privileges** = the service’s filesystem write permissions (typically high).  
**Destination** = the arbitrary file location on the target system.

## Key Concepts

### Directory Traversal (Path Traversal)
- A technique that uses `../` (dot‑dot‑slash) sequences to move up directories.
- In this vulnerability, `../../../../../../whoops` climbs up enough levels to reach the root (`C:\`) and then creates a file named `whoops`.
- The `--path-as-is` flag in curl prevents the client from normalizing `../` sequences (some clients would strip them).

### Arbitrary File Write
- After bypassing the path restriction, the server writes the request body (`--data-binary`) to the specified file.
- No additional validation or sandboxing occurs – the file is created with the service’s privileges.

### The Attack Cycle (as described in the module)

| Step | Directory Traversal Phase | Concept Category |
|------|--------------------------|------------------|
| 1 | User specifies PUT request, traversal path, and file content | Source |
| 2 | The HTTP request processor handles the path and content | Process |
| 3 | The application checks authorization only for the base folder; traversal bypasses it | Privileges |
| 4 | The request is forwarded to a local file‑writing process | Destination (local) |

| Step | Arbitrary File Write Phase | Concept Category |
|------|---------------------------|------------------|
| 5 | Same user input (filename + content) becomes the source for the write operation | Source |
| 6 | The write process takes the filename and content | Process |
| 7 | Because restrictions were bypassed, the write is approved | Privileges |
| 8 | The file is written to the specified location on the local system | Destination |

## Why It Matters
- This vulnerability shows that **authentication does not guarantee safety**. Even with valid credentials, a service can be severely compromised.
- The attack uses **HTTP PUT** – a method often forgotten or misconfigured. Many administrators test FTP but leave the web interface exposed on port 443 with default credentials.
- Directory traversal vulnerabilities are still common in enterprise software. They turn a limited file write into full system compromise.
- For penetration testers, this is a high‑impact finding: arbitrary file write often leads directly to RCE (e.g., writing a `.aspx` webshell into `C:\inetpub\wwwroot`).

## Defender Perspective
- **Detection:** Monitor HTTP logs for `PUT` requests containing `../` or `..\` sequences. Look for `--path-as-is` behavior (raw URI not normalized). Also monitor for unexpected file creations outside the web root (File Integrity Monitoring).
- **Mitigation:**  
  - Upgrade CoreFTP to build 727 or later.  
  - Disable HTTP/HTTPS management interface if not needed.  
  - Run the service with least privilege (not SYSTEM).  
  - Use a Web Application Firewall (WAF) to block path traversal patterns.  
  - Implement strict input validation on file paths – reject any request containing `..` or URL‑encoded variants.
- **MITRE ATT&CK:** T1006 (Directory Traversal), T1222 (File Permissions Modification – if writing to sensitive locations), T1105 (Ingress Tool Transfer – uploading malicious files).

## Key Takeaways
- **Always test for path traversal** even when you have authenticated access. Authentication is not a magic shield.
- **Try unusual HTTP methods** – `PUT`, `DELETE`, `PATCH`, `OPTIONS`. Many services support them without proper security controls.
- The `--path-as-is` flag in curl is essential for traversal testing – otherwise curl may “help” by normalizing the path and removing the `../`.
- When you find an arbitrary file write, immediately attempt to write a webshell (`.php`, `.aspx`, `.jsp`) into a known web root. That turns file write into RCE.
- The “Concept of Attacks” pattern helps you debug why an exploit works – trace each step and see where the breakdown happens (path validation, privilege check, destination access).

## Gotchas
- The vulnerability requires **valid credentials**. You still need to brute‑force or guess a username/password first (default credentials often work).
- The service may run on a non‑standard port (e.g., 8080, 8443). Always scan all open ports and test for HTTP services.
- Some servers normalize the path **before** checking permissions – if they strip `../`, the attack fails. `--path-as-is` prevents client‑side normalization, but the server might still normalize. Test with different depths (`../../../../`) and encoding (`%2e%2e%2f`).
- Writing a file to `C:\Windows\System32\drivers\etc\hosts` or other critical locations could break the system – be careful in production assessments.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|common-services]]  
← [[04-finding-sensitive-info]] | [[07-attacking-smb]] →
<!-- AUTO-LINKS-END -->
