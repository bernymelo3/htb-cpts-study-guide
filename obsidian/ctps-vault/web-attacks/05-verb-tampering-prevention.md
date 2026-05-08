## ID
830

## Module
Web Attacks

## Kind
notes

## Title
Section 5 — Verb Tampering Prevention

## Description
Explains how to secure web servers and application code against HTTP Verb Tampering by using safe configuration directives and consistent HTTP method handling.

## Tags
defense, prevention, http-verb-tampering, secure-coding, server-hardening

## TL;DR — What's Important
- **Avoid per‑verb authorization:** Using `<Limit GET>` or `<http-method>GET</http-method>` leaves other methods unprotected; use `LimitExcept`, `http-method-omission`, or `add/remove` to cover all methods.
- **Never mix method‑specific filtering with method‑agnostic execution:** Filtering `$_POST` but commanding on `$_REQUEST` lets attackers bypass via GET or other verbs.
- **Consistency is key:** Apply the same validation logic to all request methods; if you must use `$_REQUEST`, validate `$_REQUEST`.
- **Disable unnecessary methods globally:** Block HEAD, PUT, DELETE at the web server level if the application doesn’t need them.
- **Code separation hides risks:** When validation and execution functions are scattered, Verb Tampering becomes harder to spot; centralize input handling.

## Concept Overview
HTTP Verb Tampering prevention requires both server‑level configuration hardening and code‑level consistency. On the server side, it’s common to restrict authorization to a specific HTTP method (e.g., `<Limit GET>`), which inadvertently leaves other methods unprotected. On the code side, a security filter might check `$_POST` while the subsequent command uses `$_REQUEST`, allowing a malicious GET parameter to slip past the filter. The section provides vulnerable configuration examples for Apache, Tomcat, and ASP.NET, and a PHP code snippet that demonstrates the inconsistency. It then shows how to apply safe alternatives and enforce method‑agnostic input handling.

## Key Concepts

### Server Configuration Hardening
- **Apache:** Replace `<Limit GET>` with `<LimitExcept GET POST>` (or `LimitExcept` covering all intended methods). This denies any verb that is not explicitly listed.
- **Tomcat:** Use `<http-method-omission>` inside the `<web-resource-collection>` to exclude certain methods from the security constraint, rather than `<http-method>` which only includes one.
- **ASP.NET:** Use `<deny verbs="*" users="*"/>` or dynamic checks; avoid restricting `<allow verbs="GET">` alone.
- **General:** Disable HEAD requests if not required, as they can often bypass authentication configured only for GET/POST.

### Code‑Level Prevention
- **Consistent parameter retrieval:** If validation uses `$_POST`, the execution should also use `$_POST`. If using `$_REQUEST`, validate `$_REQUEST` directly.
- **Expand filter scope:** When a filter must protect against injection, iterate over all request parameter sources (GET, POST, Cookie) or use a unified method like `$_REQUEST`.
- **Avoid method‑specific logic in separate files:** Centralize request handling so that input is always sanitized through the same pipeline, regardless of the HTTP verb.

### Method‑Agnostic Parameter Functions
| Language | Safe, All‑Methods Parameter Access |
|----------|-----------------------------------|
| PHP | `$_REQUEST['param']` (if validated) |
| Java | `request.getParameter("param")` |
| C# | `Request["param"]` |

## Why It Matters
Verb Tampering vulnerabilities are often a result of a small oversight: limiting a check to one verb while leaving the door open for others. This oversight can completely undermine authentication or re‑expose already‑patched injection bugs. Knowing how to harden configurations and write consistent request handling prevents attackers from turning a minor misconfiguration into a full compromise. For penetration testers it’s essential to not only detect these issues but also to articulate the exact fix required.

## Defender Perspective
- **Detection:** Log and alert on HTTP methods that are unusual for a given endpoint. For instance, a HEAD or PUT request to a user profile page should raise a flag.
- **Mitigations:**
  - Server: Use safe directives (`LimitExcept`, `http-method-omission`, global method filtering) and return `405 Method Not Allowed` for disallowed methods.
  - Code: Never trust that a request will arrive via a specific method; validate all sources or enforce method whitelisting in the application.
  - WAF: Configure rules to block methods not listed as allowed per application path.
- **MITRE ATT&CK:** Verb Tampering can be a step toward T1078 (Valid Accounts) by bypassing authentication, or toward T1203 (Exploitation for Client Execution) if it enables command injection.

## Key Takeaways
- Always use exclusionary directives (`LimitExcept`) instead of per‑verb inclusion to avoid accidentally leaving methods unprotected.
- `$_REQUEST` is not inherently evil—it’s the inconsistency between it and method‑specific filters that creates the vulnerability.
- In large codebases, trace the full path of a user input parameter to ensure it’s validated for every HTTP verb that can inject it.
- Disabling HEAD requests can close a common authentication‑bypass vector without breaking normal user workflows.
- As a tester, for every endpoint, send the same malicious payload with GET, POST, HEAD, PUT, DELETE, and custom verbs to verify that all methods are either blocked or properly sanitized.

## Gotchas
- Some frameworks (e.g., ASP.NET MVC) automatically map HTTP methods to controller actions; an unpredictable method might hit an entirely different (unsafe) handler.
- `LimitExcept GET POST` in Apache still allows HEAD, OPTIONS, etc. unless explicitly denied; HEAD may return the same response as GET, leaking information.
- PHP’s `$_REQUEST` order can be configured via `request_order` in php.ini; if it includes 'C', cookie values can override POST/GET and create additional bypass opportunities.
- Blocking too many methods globally may break legitimate features like health checks or CORS preflight (OPTIONS); always discuss whitelisting with the application owner.