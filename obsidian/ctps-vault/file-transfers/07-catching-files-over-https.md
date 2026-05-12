# NOTES — Catching Files over HTTP/S

## ID
621

## Module
File Transfers

## Kind
notes

## Title
Section 7 — Catching Files over HTTP/S

## Description
Configure Nginx to accept HTTP PUT uploads (without the RCE risks of an Apache + PHP misconfig) so you have a stable, encrypted-capable file catcher on TCP/443 or any port.

## Tags
nginx, http-put, upload, webdav, dav-methods, catching-files, apache-vs-nginx

## Commands
- `sudo mkdir -p /var/www/uploads/SecretUploadDirectory`
- `sudo chown -R www-data:www-data /var/www/uploads/SecretUploadDirectory`
- `sudo ln -s /etc/nginx/sites-available/upload.conf /etc/nginx/sites-enabled/`
- `sudo systemctl restart nginx.service`
- `sudo rm /etc/nginx/sites-enabled/default`
- `curl -T /etc/passwd http://localhost:9001/SecretUploadDirectory/users.txt`
- `tail -2 /var/log/nginx/error.log`
- `ss -lnpt | grep 80`

## What This Section Covers
HTTP/HTTPS is the most permissive egress channel in most enterprises, so it's worth running a dedicated catcher. Apache + PHP makes accidental RCE too easy (anything ending in `.php` may execute). Nginx is preferred — PHP execution requires explicit configuration, so an upload dir is just an upload dir.

## Nginx PUT Upload Server — Setup

### 1. Create directory + chown
```bash
sudo mkdir -p /var/www/uploads/SecretUploadDirectory
sudo chown -R www-data:www-data /var/www/uploads/SecretUploadDirectory
```

### 2. `/etc/nginx/sites-available/upload.conf`
```nginx
server {
    listen 9001;

    location /SecretUploadDirectory/ {
        root    /var/www/uploads;
        dav_methods PUT;
    }
}
```

### 3. Enable + restart
```bash
sudo ln -s /etc/nginx/sites-available/upload.conf /etc/nginx/sites-enabled/
sudo systemctl restart nginx.service
```

### 4. Troubleshooting common issues

**Port 80 conflict on Pwnbox** (`bind() to 0.0.0.0:80 failed (98: Address already in use)`):
```bash
tail -2 /var/log/nginx/error.log
ss -lnpt | grep 80
# Pwnbox's noVNC websockify already holds :80. Remove default site:
sudo rm /etc/nginx/sites-enabled/default
```

> Pick a non-conflicting port in `upload.conf` (e.g., 9001) to avoid the Pwnbox websockify collision in the first place.

## Testing the Upload

```bash
curl -T /etc/passwd http://localhost:9001/SecretUploadDirectory/users.txt
sudo tail -1 /var/www/uploads/SecretUploadDirectory/users.txt
```

The `-T` (uppercase) flag tells curl to PUT the local file as the named URL path.

## Hardening Checklist

| Issue | Fix |
|-------|-----|
| Directory listing exposed | Nginx doesn't list by default — verify by browsing `http://host/SecretUploadDirectory/` (should be 403/404). |
| Files world-readable | Owner `www-data` only; clients shouldn't be able to download what others uploaded. |
| HTTPS for sensitive loot | Add `listen 443 ssl` + cert config. Or front with stunnel. |
| Anyone can PUT | Add `auth_basic` or restrict by source IP — otherwise the path is a wide-open dropbox. |

## Apache vs Nginx for Upload Servers

| | Apache | Nginx |
|---|--------|-------|
| PHP auto-execution | ✓ (with mod_php — dangerous if you accept `.php` uploads) | ✗ (needs explicit FastCGI config) |
| Directory listing default | ✓ on dirs without `index.*` | ✗ (must be enabled) |
| WebDAV / PUT config | `mod_dav` — more involved | `dav_methods PUT;` — one line |

## Key Takeaways
- **Use Nginx, not Apache**, for upload catchers — Apache's PHP module is a foot-gun if you ever upload `.php`.
- Nginx doesn't show directory listings by default; Apache does — Apache leaks loot to anyone who guesses the dir.
- The single line `dav_methods PUT;` is all you need to enable HTTP PUT — no full WebDAV stack required.
- Always run on **HTTPS** when handling real loot; combine with file-level encryption (see [[06-protected-file-transfers]]).

## Gotchas
- Pwnbox websockify holds TCP/80 — bind your listener to another port or remove the default site.
- `curl -t` (lowercase) is a different flag (telnet). The upload flag is `-T` (uppercase).
- Without `auth_basic` or IP restrictions, anyone who knows the URL can PUT — assume the dir name will be found via wordlist scanning.
- WebDAV `PUT` is **not authenticated** by default in this minimal config — anyone can overwrite files. Add auth before pointing real targets at it.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-transfers]]
← [[06-protected-file-transfers]] | [[08-living-off-the-land]] →
<!-- AUTO-LINKS-END -->
