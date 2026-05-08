## ID
<!-- Pick the next free number in the module's range. See CLAUDE.md → "ID Ranges by Module". -->
601

## Module
Documentation & Reporting

## Kind
notes

## Title
Section 2 — Notetaking & Organization

## Description
Covers the minimum required notetaking structure, logging setup with Tmux, evidence collection standards, folder organization, and screenshot/redaction best practices for professional penetration test assessments.

## Tags
documentation, reporting, tmux, organization, evidence, notetaking

## Commands
- `git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm`
- `touch ~/.tmux.conf`
- `tmux source ~/.tmux.conf`
- `tmux new -s <SESSION_NAME>`
- `mkdir -p <CLIENT>-IPT/{Admin,Deliverables,Evidence/{Findings,Scans/{Vuln,Service,Web,'AD Enumeration'},Notes,OSINT,Wireless,'Logging output','Misc Files'},Retest}`
- `xfreerdp /v:<STMIP> /u:htb-student /p:HTB_@cademy_stdnt!`

## What This Section Covers
Defines the minimum notetaking structure needed for professional penetration test engagements, including categories like credentials, findings, activity logs, and payload logs. Introduces Tmux logging as the go-to session recording tool and establishes a repeatable folder structure for organizing all assessment artifacts. Also covers evidence formatting, screenshot redaction best practices, and what data should never be archived.

## Methodology

### Tmux Logging Setup
1. Clone TPM: `git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm`
2. Create config: `touch ~/.tmux.conf`
3. Add the following to `~/.tmux.conf`:
```
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-sensible'
set -g @plugin 'tmux-plugins/tmux-logging'
set -g history-limit 50000
run '~/.tmux/plugins/tpm/tpm'
```
4. Apply config: `tmux source ~/.tmux.conf`
5. Start a new session: `tmux new -s <SESSION_NAME>`
6. Install plugins: `[Ctrl] + [B]` → `[Shift] + [I]`
7. Start logging: `[Ctrl] + [B]` → `[Shift] + [P]`  *(repeat to stop)*

### Key Tmux Key Bindings
| Action | Keys |
|---|---|
| Install plugins | `[Ctrl] + [B] + [Shift] + [I]` |
| Toggle logging on/off | `[Ctrl] + [B] + [Shift] + [P]` |
| Retroactive save (entire pane) | `[Ctrl] + [B] + [Alt] + [Shift] + [P]` |
| Capture current pane to file | `[Ctrl] + [B] + [Alt] + [P]` |
| Clear pane history | `[Ctrl] + [B] + [Alt] + [C]` |
| Split panes **vertically** | `[Ctrl] + [B] + [Shift] + [%]` |
| Split panes horizontally | `[Ctrl] + [B] + [Shift] + ["]` |
| Move between panes | `[Ctrl] + [B] + [O]` |

### Assessment Folder Setup
```bash
mkdir -p <CLIENT>-IPT/{Admin,Deliverables,Evidence/{Findings,Scans/{Vuln,Service,Web,'AD Enumeration'},Notes,OSINT,Wireless,'Logging output','Misc Files'},Retest}
```

## Notetaking Structure — Minimum Required Categories
| Category | Purpose |
|---|---|
| Attack Path | Full compromise chain with screenshots and command output |
| Credentials | Centralized store for all captured creds and secrets |
| Findings | One subfolder per finding with narrative + evidence |
| Vulnerability Scan Research | What you've tried, to avoid duplicate work |
| Service Enumeration Research | Services investigated, failed exploits, promising leads |
| Web Application Research | Subdomains found, interesting apps, default creds tried |
| AD Enumeration Research | Step-by-step AD enumeration log, areas of interest |
| OSINT | Interesting external data collected |
| Administrative Information | Contacts, RoE objectives, to-do list |
| Scoping Information | In-scope IPs, URLs, provided creds — quick reference |
| Activity Log | High-level timeline of everything done |
| Payload Log | Payload used, target host, file path, hash, cleanup status |

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| What tool can make logging a session easier? | Tmux | Covered in the Logging section of this module |
| How does Steve split panes vertically? | `[Ctrl] + [B] + [Shift] + [%]` | Tmux pane splitting key binding |

## Key Takeaways
- **Tmux logging is non-negotiable** — enable it at the start of every session; losing logs mid-assessment is embarrassing and unprofessional.
- Set `history-limit 50000` in `.tmux.conf` before you need retroactive logging, not after — default limit will truncate early session data.
- **Payload Log must be maintained in real time** — track host, path, hash, and cleanup status for every payload deployed; clients will ask.
- **Never blur/pixelate to redact** — use solid black bars edited directly into the image; pixelation can be reversed with tools like Unredacter.
- Use **terminal text output over screenshots** whenever possible — easier to redact, easier to highlight, smaller file size, and client can copy/paste to reproduce.
- **Strip formatting before pasting terminal output** into Word — embedded smart quotes or non-UTF-8 chars will silently break commands when the client tries to reproduce them.
- Do **not** open or extract sensitive files (PII, legal docs) from shares — screenshot the directory listing only; accessing the actual files may trigger GDPR/compliance obligations.

## Gotchas
- Retroactive logging (`[Ctrl] + [B] + [Alt] + [Shift] + [P]`) only captures what's still in the Tmux scrollback buffer — if `history-limit` is at default, you will lose early session data.
- Applying a black shape *on top of text in MS Word* is not true redaction — someone can delete the shape and read the text. Always edit the image file directly before inserting it.
- Copying text from one pane in a split Tmux window grabs text from adjacent panes too — use the pane capture shortcut (`[Ctrl] + [B] + [Alt] + [P]`) to get clean single-pane output.
