# 🔁 Phase-Generator Prompt (reuse for every phase)

The pentest phases I want (NOT a custom split):

1. **Reconhecimento (Footprinting)** — public info, domains, active servers
2. **Enumeração (Scanning)** — ports, services, exact versions
3. **Exploração (Exploitation)** — break in, first access
4. **Escala de Privilégios (Privilege Escalation)** — user → admin/root
5. **Movimentação Lateral (Lateral Movement)** — pivot to neighbouring hosts

> Phases 1+2 are already built → `01-ENUMERATION.md`.

---

## 📋 The prompt — copy, fill `{{PHASE}}` + `{{SOURCE FOLDERS}}`, paste

```
Build the PLAYBOOK file for phase: {{PHASE}}.

Format — EXACTLY like PLAYBOOK/01-ENUMERATION.md:
- One "## 🔹 <thing>" block per attack/target/technique in this phase.
- Each block = a checklist where EVERY "- [ ]" point carries its own
  command inline in backticks (multi-line → fenced block under the point).
- Commands MUST come from my own vault notes' "## Commands" sections
  (CPTS/obsidian/ctps-vault/{{SOURCE FOLDERS}}). My commands beat textbook
  defaults — quote mine, keep my paths (/opt/useful/seclists), my flags.
- Vars: $IP, $DOMAIN, $ATTACKER (don't hardcode IPs).
- After each block: "📓" line linking the deep notes with [[../folder/file]].
- Collapsible "<details><summary>▸ lab refs</summary>…</details>" with the
  module's lab/skills-assessment notes.
- End with a "## ➡️ Where next" router to the other phases.
- Order blocks simplest-first (defaults/known-CVE before clever exploits).
- Keep prose minimal — this is an operational checklist, not a study doc.

Pull commands from my notes first (grep the ## Commands sections), tell me
if a note has no command for a point, don't invent one. Save to
PLAYBOOK/<NN>-<NAME>.md, then update the task list + SEARCH.md.
```

---

## 🎯 Ready-to-fire fill-ins (just say the line)

| Say this | Builds | `{{SOURCE FOLDERS}}` |
|---|---|---|
| **"do phase 3 web"** | `02-EXPLOITATION-WEB.md` ✅ built | web-recon, ffuf, web-proxies, file-inclusion, sql-injection-fundamentals, sqlmap-fundamentals, xss, file-uploads, command-injetions, web-attacks, login-brute-forcing, attacking-common-applications |
| **"do phase 3 services"** | `03-EXPLOITATION-SERVICES.md` + shells | common-services, shells-payloads, using-the-metasploit |
| **"do phase 4 linux"** | `04-PRIVESC-LINUX.md` | linux-privallege-escalation, password-attacks (linux) |
| **"do phase 4 windows"** | `05-PRIVESC-WINDOWS.md` | windows-privesc, password-attacks (windows) |
| **"do phase 5"** | `06-LATERAL.md` | ad-enum-attacks, pivoting-tunneling, password-attacks (PtH/PtT) |
| **"do report"** | `07-REPORT.md` | documentation (Bruno trigger method) |

> **Filename = sequential, one number per file, no gaps** (recon+scan share `01`; web `02`; services `03`; linux `04`; windows `05`; lateral `06`; report `07`).
> Fire them one at a time so each file gets full attention. `00-INDEX.md` is
> already aligned to this; SEARCH reindex is the final step after all files exist.
