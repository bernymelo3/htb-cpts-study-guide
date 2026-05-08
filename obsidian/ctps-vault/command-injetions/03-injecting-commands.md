# Section 3 — Injecting Commands

## ID
601

## Module
Command Injections

## Kind
notes

## Title
Section 3 — Injecting Commands

## Description
Bypasses front-end input validation via Burp Suite to inject OS commands through the Host Checker's ping functionality, confirming back-end command injection.

## Tags
command-injection, burp-suite, front-end-bypass, url-encoding, input-validation

## Commands
- 127.0.0.1; whoami
- ping -c 1 127.0.0.1; whoami

## What This Section Covers
Front-end input validation (HTML pattern attributes) can block injection payloads in the browser, but offers zero protection if the back-end doesn't also validate. By intercepting the request with a proxy like Burp Suite, you bypass the client-side check entirely and send arbitrary payloads straight to the server.

## Methodology
1. Attempt injection directly in the browser (`127.0.0.1; whoami`) — observe it's blocked client-side with "Please match the requested format."
2. Open DevTools (`Ctrl+Shift+E`, Network tab) and confirm no HTTP request was sent — proof the validation is front-end only.
3. View page source (`Ctrl+U`) and find the `pattern` regex attribute on the input field (line 17).
4. Configure Burp Suite / ZAP as a proxy, intercept a normal request with a valid IP (`127.0.0.1`).
5. Send the intercepted request to Repeater (`Ctrl+R`).
6. Replace the IP value with the payload `127.0.0.1; whoami`, URL-encode it (`Ctrl+U`), and click Send.
7. Confirm the response contains both the ping output and the `whoami` result — command injection confirmed.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Review the HTML source code to find where front-end input validation is happening. On which line number? | **17** | `Ctrl+U` to view source — regex `pattern` attribute on the input field is on line 17 |

## Key Takeaways
- Front-end validation is a UX convenience, not a security control — always test by sending requests directly to the back-end.
- No network request on submit = client-side validation blocking you, not the server.
- Burp Repeater lets you freely modify and resend requests, bypassing any browser-side restrictions.
- Always URL-encode injection payloads (`Ctrl+U` in Burp) to avoid breaking the HTTP request structure (especially `&`, `;`, `|`).

## Gotchas
- If you forget to URL-encode, characters like `&` will be interpreted as HTTP parameter separators instead of shell operators.
- The `pattern` attribute only enforces validation in the browser — `curl` or Burp ignore it completely.
