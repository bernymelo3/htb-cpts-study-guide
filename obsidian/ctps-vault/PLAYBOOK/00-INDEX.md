# 🎯 THE PLAYBOOK — Index

> Open the phase you're in. Each file = **checklist → commands → link to deep notes → (collapsible) lab refs**.
> The commands are right there. You only "ask Claude" when stuck, not to find a command.

```
   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │ 01 ENUMERATE │ → │ 02 EXPLOIT   │ → │ 04 PRIVESC   │ → │ 06 LATERAL   │
   │ recon + scan │   │ 02 web       │   │ 04 linux     │   │ AD · pivot · │
   │ ports+svcs   │   │ 03 services  │   │ 05 windows   │   │ PtH/PtT      │
   └──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
          ▲                                                         │
          └──────────────  new host / creds → back to 01  ──────────┘
                                                  │
                          ┌──────────┐            │
   every finding, always ─→ │ 07 REPORT │ ←────────┘
                          └──────────┘
```

> **Sequential, one file per number — no gaps.** Recon + scanning share `01`. Exploitation splits into `02` web / `03` services; privesc into `04` linux / `05` windows. Lateral (AD + pivoting + PtH/PtT) is `06`. Report `07` is cross-cutting — do it the moment you confirm a finding.

## Phase files

| #   | Phase                                | You're here when…                                                    | File                           |
| --- | ------------------------------------ | -------------------------------------------------------------------- | ------------------------------ |
| 01  | **Enumeration** (recon + scan)       | got an IP/host, need to know what's on it                            | `[[01-ENUMERATION]]`           |
| 02  | **Exploitation — Web**               | port 80/443/8080 — a website                                         | `[[02-EXPLOITATION-WEB]]`      |
| 03  | **Exploitation — Services + Shells** | non-web service, or have RCE & need a stable shell                   | `[[03-EXPLOITATION-SERVICES]]` |
| 04  | **Privesc — Linux**                  | low-priv shell on Linux                                              | `[[04-PRIVESC-LINUX]]`         |
| 05  | **Privesc — Windows**                | low-priv shell on Windows                                            | `[[05-PRIVESC-WINDOWS]]`       |
| 06  | **Lateral movement**                 | domain/DC in scope, or can't reach a subnet, or have a hash to spray | `[[06-LATERAL]]`               |
| 07  | **Report**                           | confirmed ANY finding (do it NOW)                                    | `[[07-REPORT]]`                |

> Files marked `[[…]]` for phases not yet built are placeholders until that phase is generated (`_PROMPT.md`). SEARCH reindex is the final step after all phases exist.

## The only 3 rules

1. **Enumerate fully before exploiting.** Every port, every service, every input.
2. **Simplest first.** Anon login / default creds / known CVE before clever exploits.
3. **Stuck 15–20 min → it's the wrong path, not a hard one.** Next item on the checklist. Ask Claude with *exact output*.

## Set these once per engagement

```bash
export IP=10.129.x.x          # current target
export ATTACKER=10.10.14.x    # your tun0 — `ip a show tun0`
export DOMAIN=inlanefreight.local
```
Keep open: `creds.txt` · `hosts.txt` · `flags.txt` · `loot/`

> Macro 10-day loop & time-boxing → `[[../00-EXAM-MASTER]]`. Symptom lookup → `[[../ATTACK-PATHS]]`. This PLAYBOOK is the operational layer; deep per-technique notes are linked inside each phase.
