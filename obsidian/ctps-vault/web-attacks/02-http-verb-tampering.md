## ID
810

## Module
Web Attacks

## Kind
notes

## Title
Section 2 — Intro to HTTP Verb Tampering

## Description
Introduces HTTP Verb Tampering attacks, explaining how over‑permissive server configurations and inconsistent handling of HTTP methods can lead to authentication bypass or filter evasion.

## Tags
theory, concept, http-verb-tampering, authentication-bypass, web-attacks

## TL;DR — What's Important
- **HTTP supports more than GET/POST:** HEAD, PUT, DELETE, OPTIONS, PATCH — each can trigger unintended functionality if not properly restricted.
- **Verb Tampering can bypass authentication:** If server limits login checks to only GET and POST, a HEAD or PUT request may access pages unauthenticated.
- **Inconsistent method handling is a code‑level risk:** Using `$_REQUEST` (which merges GET, POST, COOKIE) while filtering only one verb can reopen injection vulnerabilities.
- **Fix it with explicit allow‑listing:** Configure servers to accept only the HTTP methods actually needed per resource, and validate input consistently across all request methods.

## Concept Overview
HTTP Verb Tampering exploits the fact that web servers and applications often focus on the two most common methods (GET and POST) while ignoring other valid HTTP verbs. If the web server's access control or the application’s input filters are applied to only a subset of verbs, an attacker can send the same request with a different method — bypassing authentication checks or circumventing security filters. The vulnerability can arise from either a server misconfiguration (like `<Limit>` directives that miss verbs) or from sloppy coding (filtering one method but using a combined request variable in the code).

## Key Concepts

### The HTTP Methods at Play
| Method | Intended Purpose | Risk if Unrestricted |
|--------|------------------|----------------------|
| HEAD | Like GET but returns only headers | Can bypass authentication if not included in `<Limit>` |
| PUT | Writes payload to a specified URL | May allow uploading a web shell or overwriting files |
| DELETE | Removes the specified resource | Could delete critical files or configuration |
| OPTIONS | Lists allowed methods | Information disclosure; reveals attack surface |
| PATCH | Partially modifies a resource | May alter data if not properly guarded |

### Two Vulnerability Types
1. **Server Misconfiguration**  
   Example Apache configuration:  
   ```xml
   <Limit GET POST>
       Require valid-user
   </Limit>