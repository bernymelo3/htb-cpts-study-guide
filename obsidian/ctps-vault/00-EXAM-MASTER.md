# CPTS Exam — Master Kill-Chain (L0 spine)

**What this is:** the *one* doc you keep open. It is NOT a technique reference — it's the macro loop + where flags live + when to abandon a path. It drives the other two layers, it does not duplicate them.

- **L0 (this doc)** — the engagement loop + time-boxes + flag accounting.
- **L1 — `ATTACK-PATHS.md`** — "I have X → go to note Y" (symptom router).
- **L2 — `<module>/00-METHODOLOGY.md`** — depth, exact commands.

> Rule: when this doc says "→ ATTACK-PATHS §N", you jump there. Never freestyle. The exam is won by *enumeration discipline and not getting stuck*, not by cleverness.

**Exam shape:** one blackbox AD-centric enterprise network. ~12–14 flags scattered across hosts, behind pivots and (often) a domain trust. 10 days: ~7 hacking, **≥1.5 days report** (the report is what fails people). 85% (≈10–12/14 + a passing report) to pass.

---

## The macro loop (this is the whole exam)

```
SCOPE  →  EXTERNAL FOOTHOLD  →  ┌─────────────── PER-HOST LOOP ───────────────┐
                                │  enum → exploit → shell → STABILIZE →        │
                                │  LOOT → privesc → LOOT AGAIN → flag-sweep    │
                                │  → feed creds/hosts back into pool           │
                                └───────────────┬──────────────────────────────┘
                                                │ new host / new creds?
                                  PIVOT ◄────────┘  yes → re-enter PER-HOST LOOP
                                    │
                                    ▼
                              AD PATH → DA → DCSYNC → trust hop → repeat loop
                                    │
                                    ▼
                            14 flags accounted for? → REPORT
```

You are never "doing a box." You are running the per-host loop on whatever host the credential/pivot pool currently lets you reach. **Every new credential is sprayed against every known host before you escalate anything.** That single habit finds more flags than any exploit.

---

## Phase 0 — Scope & setup (15 min, once)

- Read the scope letter. Note in-scope CIDRs, the **assumed-breach** start point (if any), excluded hosts, flag format.
- `creds.txt`, `hosts.txt`, `flags.txt`, `tunnels.txt` open from minute one. Per-host folder `host-<ip>/` with screenshot subfolder. → format rules in **Daily discipline** below.
- Pre-exam checklist already done? If not → `EXAM-WARNINGS.md` §8.

## Phase 1 — External foothold

1. Full scan **every** in-scope host, don't stop at top-1000. → **ATTACK-PATHS §1e** (nmap) — kick `-p- -sV --min-rate 1000` in background, start working the fast hits.
2. For each open port → **ATTACK-PATHS §2** (port→service router). Web → **§1b/§1c/§3**. Service → **§1d** (footprinting per-service block).
3. Goal of Phase 1 = **one shell or one valid credential inside the perimeter.** The moment you have it, STOP widening externally and enter the per-host loop on what you got.

## Phase 2 — Per-host loop (run on EVERY host you touch, no exceptions)

Do these in order. Do not skip LOOT to chase privesc — creds beat exploits.

| # | Step | Route |
|---|---|---|
| 1 | **Stabilize shell first** (TTY upgrade) — before anything else, or you lose it | ATTACK-PATHS §3b |
| 2 | **Identity dump:** `id`/`whoami /all`, hostname, OS, `ip a`/`ipconfig /all` (← 2nd NIC = pivot signal) | — |
| 3 | **LOOT pass #1 (pre-privesc):** bash/PS history, `~/.ssh`, `/etc/`, configs, DB creds, web.config, mounted shares, `C:\Users\*` | ATTACK-PATHS §7 (cred hunting) |
| 4 | **Privesc** → root/SYSTEM | Linux ATTACK-PATHS §5c · Windows §5b |
| 5 | **LOOT pass #2 (post-privesc):** SAM/LSASS/shadow, SSH keys, `secretsdump`, browser/app creds | ATTACK-PATHS §7 |
| 6 | **Flag sweep this host** (see flag-location checklist ↓) → log to `flags.txt` immediately |
| 7 | **Feed back:** every new cred/hash/key → spray across ALL hosts in `hosts.txt`; every new host/subnet → `hosts.txt` | ATTACK-PATHS §7 (don't crack what you can pass) |
| 8 | **Pivot?** 2nd NIC or new subnet → set up tunnel NOW | ATTACK-PATHS §6 |

Then re-enter the loop on the next reachable host. Repeat until the credential/host pool stops growing.

## Phase 3 — AD path

Once you have any domain credential or a domain-joined host:

- → **ATTACK-PATHS §5** ("AD by what you have"). Run BloodHound early even with low creds.
- Standing order: every domain cred → check Kerberoast + AS-REP + ACLs + delegations + trusts. Chains hide in attributes you didn't query.
- Goal ladder: domain user → privileged user → local admin on a box → **DCSync / NTDS** → DA → **trust hop** (the exam usually has a second domain/forest behind a trust — if you've been stuck in one domain for hours, `Get-DomainTrust`/`nltest` → ATTACK-PATHS §5 trust rows).

## Phase 4 — Flag accounting → report

Stop hacking when `flags.txt` ≈ target **or** you hit the report time-box, whichever is first. → `documentation/` + ATTACK-PATHS §8.

---

## Where flags physically live (sweep every host)

- `C:\Users\<user>\Desktop\flag.txt`, `\Documents\`, Administrator desktop
- `/home/<user>/`, `/root/`, `/var/www/`, web app DB rows, app config
- SMB shares, FTP roots, mounted NFS
- Inside DBs (`SELECT` after SQLi / DB creds), inside KeePass/`.kdbx`
- On the **DC** (after DCSync/NTDS — high-value flag often here)
- In credential material itself (a flag string used as a password/note)

If a host has no obvious flag, you're not done enumerating it — re-loot, don't move on.

---

## Time-boxes (the rule that actually saves the exam)

| Situation | Hard limit | Then |
|---|---|---|
| One exploit/vuln class not yielding | **60–90 min** | switch vector — ATTACK-PATHS by current state |
| Privesc on one host | **45 min** of structured checks | run linpeas/winpeas, re-check `sudo -l`/`whoami /priv`, then move on and come back |
| Cracking one hash | if not falling to rockyou+best64 fast | **stop** — pass-the-hash / ACL / other vector instead |
| Stuck overall | **20 min** | run the **"I am stuck" protocol** ↓ — no skipping steps |
| Custom exploit dev | only in last 24h of hacking, everything else ruled out | otherwise it's quicksand |
| **Start the report** | when **≤ 36h** remain | non-negotiable; ≤24h = you finish but miserable |

## "I am stuck" protocol (in order, no skipping)

1. Re-read your last 30 min of notes — dropped thread?
2. `ATTACK-PATHS.md` search by current state.
3. Full `-p- -sV` if you only ran top-1000.
4. Enumerate ONE surface you skipped: vhosts, subdomains, UDP, SNMP, NFS, LDAP anon bind, IPv6.
5. Re-loot the **last host you owned** — `~/.bash_history`, `~/.config/`, `/etc/`, shares.
6. 10 min away from screen.
7. Dump full state (ports/services/creds/shells/tried) to AI → "what enumeration step did I plausibly skip?"

Never write custom exploits before step 7 + last-24h.

---

## Daily discipline (this is the report's raw material — do it live)

- **Notes are report-grade in real time:** "*The tester ran `<exact cmd>` and received:*" + screenshot. Re-running exploits later for evidence is the #1 time-sink.
- After **every** shell, dump to `host-<ip>/recon.txt`: hostname, whoami, ip, OS, users, groups, shares, processes.
- `creds.txt` line format: `proto://user:pass@host:port  (source: <where found>)` — this is finding evidence verbatim.
- Screenshots immediately, more than you need; cover secrets with a **solid** rectangle (not blur).
- `git commit` notes every ~2h. Exfil loot off Pwnbox continuously (it dies day 4).
- Hard personal rules: no hacking past 1 AM (2 AM bugs cost 2h at 9 AM); sleep ≥6h before report-eve; **always submit a report even if you didn't pass** (no report = no retake).
- Report mechanics, findings workflow, exec-summary recipe, submission limits (PDF/ZIP, no password, ≤20 MB, English) → `EXAM-WARNINGS.md` §6/§7 + `documentation/`.

---

**Last updated:** 2026-05-16 · pairs with `ATTACK-PATHS.md` (L1) and `<module>/00-METHODOLOGY.md` (L2).
