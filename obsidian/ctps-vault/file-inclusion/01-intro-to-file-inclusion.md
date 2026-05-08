## ID
500

## Module
File Inclusion

## Kind
notes

## Title
Section 1 — Intro to File Inclusions

## Description
Introduces Local File Inclusion (LFI) vulnerabilities, how they manifest across different web stacks (PHP, NodeJS, Java, .NET), and the critical distinction between reading and executing files.

## Tags
theory, file-inclusion, lfi, php, nodejs, java, dotnet

## TL;DR — What's Important
- **LFI occurs when user input controls a file path that is then loaded or executed** – most often in dynamic page‑loading mechanisms like `?page=about`.
- **It affects nearly every back‑end language** – PHP, NodeJS, Java, .NET, and others all have functions that include local files based on parameters.
- **The function used determines the impact:** read‑only functions expose source code and data; execute functions can lead to remote code execution.
- **Remote URL inclusion is also possible with some functions** – which may allow attackers to fetch and execute remote payloads.
- **Even read‑only LFI can be devastating** – source code disclosure can reveal credentials, API keys, or other vulnerabilities.
- **All user‑supplied paths must be validated and restricted** – never allow `../`, absolute paths, or remote URLs without strict whitelisting.

## Concept Overview
Local File Inclusion (LFI) is a vulnerability that allows an attacker to read (and sometimes execute) arbitrary files on the server. It arises when a web application uses user‑controlled input to determine which file to load, typically for templating, language switching, or page routing. By manipulating parameters like `?page=`, attackers can traverse directories and include sensitive files (e.g., `/etc/passwd`, source code, configuration files). When the function also executes code (e.g., PHP `include()`), the attacker can escalate to remote code execution.

## Key Concepts

### How LFI Works
- A parameter (e.g., `language`, `page`) specifies a file to include.
- No sanitization → attacker can inject `../` sequences to escape intended directories.
- Example: `?language=../../../../etc/passwd` would show the system password file if the inclusion point is deep enough.

### Vulnerable Code Patterns Across Languages

| Language | Example Vulnerable Code |
|----------|--------------------------|
| PHP | `include($_GET['language']);` |
| Node.js | `fs.readFile(path.join(__dirname, req.query.language), ...)` |
| Java | `<jsp:include file="<%= request.getParameter('language') %>" />` |
| .NET | `Response.WriteFile(HttpContext.Request.Query['language']);` |

Other PHP functions that can be dangerous: `include_once()`, `require()`, `require_once()`, `file_get_contents()`, `fopen()`. In NodeJS: `res.render()` with dynamic path. In .NET: `@Html.Partial()` with user input.

### Read vs. Execute (and Remote URL support)
| Function | Reads Content | Executes Code | Remote URL Allowed |
|----------|---------------|----------------|---------------------|
| **PHP** `include()`/`include_once()` | ✅ | ✅ | ✅ |
| **PHP** `require()`/`require_once()` | ✅ | ✅ | ❌ |
| **PHP** `file_get_contents()` | ✅ | ❌ | ✅ |
| **PHP** `fopen()`/`file()` | ✅ | ❌ | ❌ |
| **NodeJS** `fs.readFile()` | ✅ | ❌ | ❌ |
| **NodeJS** `fs.sendFile()` | ✅ | ❌ | ❌ |
| **NodeJS** `res.render()` | ✅ | ✅ (template execution) | ❌ |
| **Java** `include` | ✅ | ❌ | ❌ |
| **Java** `import` | ✅ | ✅ | ✅ |
| **.NET** `@Html.Partial()` | ✅ | ❌ | ❌ |
| **.NET** `@Html.RemotePartial()` | ✅ | ❌ | ✅ |
| **.NET** `Response.WriteFile()` | ✅ | ❌ | ❌ |
| **.NET** `include` | ✅ | ✅ | ✅ |

**Reading** functions expose file contents (source code, credentials, logs).  
**Executing** functions can run shell commands if attacker uploads a malicious file or injects into logs.  
**Remote URL** support opens Server‑Side Request Forgery (SSRF) and potential Remote File Inclusion (RFI).

## Why It Matters
LFI is one of the most impactful web vulnerabilities. Even if the initial access is only file read, source code disclosure often yields database credentials, API keys, or session tokens that enable full system compromise. When execution is possible, the attacker can gain a command shell on the server. Because dynamic page loading is extremely common, LFI vulnerabilities appear frequently in web applications.

## Defender Perspective
- **Detection:** Monitor for `../`, `/etc/passwd`, `file://`, `php://` in URL parameters and server logs. WAFs can block common traversal patterns.
- **Mitigations:**
  - **Input validation:** whitelist allowed filenames or paths; never accept raw paths.
  - **Restrict inclusion** to a specific directory and strip directory separators.
  - **Disable remote includes** (`allow_url_include=Off` in PHP).
  - **Use static file serving** when possible instead of dynamic inclusion.
  - **Limit file permissions** so the web server user cannot read sensitive files.
- **MITRE ATT&CK:** *T1082 – System Information Discovery* (file read), *T1003 – OS Credential Dumping* (reading shadow/passwd), *T1059 – Command and Scripting Interpreter* (if RCE is achieved).

## Key Takeaways
- LFI is not PHP‑only; every major web framework has a potential inclusion vulnerability if user input controls file paths.
- The function table is crucial for assessing impact: a read‑only LFI in `file_get_contents` is less severe than an executable LFI in `include`.
- Remote File Inclusion (RFI) is a subset where the attacker includes a remote script; many modern PHP configurations disable this by default, but it still appears in legacy apps.
- Path traversal depth depends on the application’s directory structure; test with many `../` sequences to reach root.
- Even if direct LFI fails, log poisoning (injecting PHP code into access logs and including them) can turn a read‑only LFI into RCE.

## Gotchas
- Windows servers use backslashes, but `..\` traversal also works; test both.
- Some frameworks sanitize `../` but allow absolute paths (`/etc/passwd`) — always try absolute paths.
- URL encoding (`..%2F`), double encoding (`..%252F`), and null bytes (`%00`) can bypass naive filters.
- The `include()` remote URL capability can be used even without RFI to perform SSRF by requesting internal services.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-inclusion]]  
[[02-lfi-basics]] →
<!-- AUTO-LINKS-END -->
