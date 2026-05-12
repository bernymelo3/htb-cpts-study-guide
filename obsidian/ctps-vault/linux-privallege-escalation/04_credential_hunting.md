# Linux Privilege Escalation — Credential Hunting

## ID
602

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 4 — Credential Hunting

## Description
Systematic search for credentials across config files, web roots, SSH keys, mail spools, backups, databases, and history files to enable lateral movement or direct privilege escalation.

## Tags
linux-privesc, credential-hunting, ssh-keys, web-config, passwords

## Commands
- `grep 'DB_USER\|DB_PASSWORD' <WP_CONFIG_PATH>`
- `find / -name "wp-config.php" 2>/dev/null`
- `find / ! -path "*/proc/*" -iname "*config*" -type f 2>/dev/null`
- `find / -name "id_rsa" -o -name "id_ecdsa" -o -name "id_ed25519" 2>/dev/null`
- `ls -la ~/.ssh/`
- `cat ~/.ssh/known_hosts`
- `grep -ri "password" /etc/ /home/ /var/www/ 2>/dev/null`
- `find / -name "*.bak" -o -name "*.backup" -o -name "*.old" 2>/dev/null`
- `find / -name "*.db" -o -name "*.sqlite" -o -name "*.sql" 2>/dev/null`
- `ls -la /var/mail/ /var/spool/mail/ 2>/dev/null`

## What This Section Covers
Credentials are everywhere on a compromised Linux host — in web application config files, bash history, SSH private keys, mail spools, backup files, and database dumps. This section teaches where to look and what patterns to grep for. Found credentials may allow escalation to another user, root access, or lateral movement to other systems in the environment.

## Methodology
1. **Web application configs** (highest-value first):
   - `find / -name "wp-config.php" 2>/dev/null` — WordPress DB credentials
   - `grep 'DB_USER\|DB_PASSWORD' <path_to_wp-config.php>`
   - Also check for other CMS configs: Drupal (`settings.php`), Joomla (`configuration.php`), Laravel (`.env`)
2. **Broad config file search**: `find / ! -path "*/proc/*" -iname "*config*" -type f 2>/dev/null` — scan output for SSH configs, database configs, application configs.
3. **Grep for passwords** across key directories:
   - `grep -r "password" /var/www/ 2>/dev/null`
   - `grep -ri "password\|secret\|key\|credential" /etc/ 2>/dev/null`
   - `grep -ri "password" /home/ 2>/dev/null`
4. **SSH private keys**:
   - `find / -name "id_rsa" -o -name "id_ecdsa" -o -name "id_ed25519" 2>/dev/null`
   - `ls -la ~/.ssh/` — check every user's `.ssh` directory
   - `cat ~/.ssh/known_hosts` — reveals lateral movement targets
5. **Mail and spool directories**: `ls -la /var/mail/ /var/spool/mail/ 2>/dev/null` — may contain password resets, internal comms with credentials.
6. **Backup files**: `find / -name "*.bak" -o -name "*.backup" -o -name "*.old" 2>/dev/null` — old configs often retain credentials.
7. **Database files**: `find / -name "*.db" -o -name "*.sqlite" -o -name "*.sql" 2>/dev/null` — SQL dumps may contain user tables with hashes.
8. **Files with "password" in the name**: `find / -iname "*password*" -type f 2>/dev/null`.
9. **History files**: `cat /home/*/.bash_history 2>/dev/null` — users pass passwords as CLI arguments.
10. **Try all found passwords** against every user on the system — password reuse is extremely common.

## Multi-step Workflow (optional)
```bash
# Full credential hunting sweep
echo "=== WordPress configs ==="
find / -name "wp-config.php" 2>/dev/null | xargs grep -l "DB_PASSWORD" 2>/dev/null

echo "=== SSH keys ==="
find / -name "id_rsa" -o -name "id_ecdsa" -o -name "id_ed25519" 2>/dev/null

echo "=== Password in files ==="
grep -ri "password" /etc/ /home/ /var/www/ 2>/dev/null | grep -v "Binary"

echo "=== Backup files ==="
find / -name "*.bak" -o -name "*.backup" -o -name "*.old" 2>/dev/null

echo "=== Database files ==="
find / -name "*.db" -o -name "*.sqlite" -o -name "*.sql" 2>/dev/null

echo "=== Mail ==="
ls -la /var/mail/ /var/spool/mail/ 2>/dev/null

echo "=== History ==="
cat /home/*/.bash_history 2>/dev/null | grep -i "pass\|secret\|key\|token"
```

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Find the WordPress database password. | *(run on target)* | `grep 'DB_USER\|DB_PASSWORD' $(find / -name "wp-config.php" 2>/dev/null)` |

> **SSH:** `htb-student` / `Academy_LLPE!` → 10.129.2.210

## Key Takeaways
- **Web roots (`/var`) are the #1 credential goldmine** — WordPress `wp-config.php`, Laravel `.env`, Drupal `settings.php` all contain DB credentials.
- **SSH keys found → immediately check `known_hosts`** — this reveals which hosts the user has connected to and enables lateral movement.
- **Password reuse is extremely common** — every password found should be sprayed against all users on the system (`su <user>`).
- **Backup files (`.bak`, `.old`)** often retain credentials from before a config was "cleaned up" — always check these.
- **Mail spools** can contain password reset emails, internal comms with credentials, or automated reports with sensitive data.
- **Bash history** catches passwords typed as CLI arguments: `mysql -u root -pSECRET`, `sshpass -p "password" ssh user@host`.
- **`/etc/shadow` requires root** — but `/etc/passwd` may contain hashes on misconfigured or embedded systems.

## Gotchas
- `grep -r "password" /var/www/` can produce massive output on large web apps — pipe through `head -50` first to gauge volume.
- Binary files will pollute grep output — add `| grep -v "Binary"` to filter.
- SSH keys may have restrictive permissions but still be readable if you share the same group — always try to `cat` them.
- WordPress may have multiple installs — `find` may return several `wp-config.php` files; check all of them.
