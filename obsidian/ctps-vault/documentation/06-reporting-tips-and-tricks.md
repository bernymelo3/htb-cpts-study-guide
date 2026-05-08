# Documentation & Reporting — Section 6
# Reporting Tips and Tricks

---

## ID
532

## Module
Documentation & Reporting

## Kind
notes

## Title
Section 6 — Reporting Tips and Tricks

## Description
Covers professional reporting workflow from MS Word best practices and macro automation, to findings databases, client communication protocol, QA processes, and final report delivery — everything needed to produce a polished, defensible pentest report end-to-end.

## Tags
reporting, documentation, ms-word, client-communication, QA, templates, findings-database

---

## Core Principle — Report as You Go

**Never leave reporting until the end of the engagement.**

- During long discovery scans → fill in templated sections (client name, scope, contacts, dates)
- During testing → write up findings with evidence as you find them
- After each day → send stop notification + high-level summary
- End of engagement → report should be ~80% done already

A rushed report = QA rejections, missed findings, and a deliverable that undersells your work.

---

## Templates

| Rule | Why |
|---|---|
| Maintain a blank template for **every** assessment type | Never recreate from scratch |
| **Never** modify a previous client's report | Risk leaving another client's name/data in the new report |
| Use placeholder variables + macros to fill common fields | Client name, dates, scope, environment names |
| Save templates as `.dotm` (not `.docx`) | Required for macro support |

> Leaving another client's name in a report = immediately amateur. Easily avoidable.

---

## MS Word — Tips & Tricks

### Platform Notes
- Use **Word for Windows** — always.
- Avoid Word for Mac: missing VB Editor, broken PDF export (trimmed margins, broken ToC hyperlinks), missing features.
- If on Mac as testing platform → spin up a Windows VM just for reporting.

---

### Font Styles
- **Never** use direct formatting (clicking Bold/Italic/Color buttons manually).
- Use named **Font Styles** for everything: headings, body, code blocks, captions.
- Why: if you update the style, **all instances** update automatically across the entire document — instead of hunting down 45 manually formatted headings.

---

### Table Styles
- Same principle as font styles — apply to all tables.
- Ensures consistency across the whole report and makes global changes trivial.

---

### Captions
- Use the built-in caption feature: **right-click image/table → Insert Caption...**
- Auto-renumbers if you add or remove figures later.
- Ties into List of Figures — won't work without proper captions.

---

### Page Numbers
- Always include — essential for client collaboration ("see paragraph 2 on page 12").

---

### Table of Contents
- Standard in every professional report.
- Default ToC is fine; customise tab leaders / page number visibility as needed.
- Update with `Ctrl+A` → `F9` (updates all fields at once — use with caution).

---

### List of Figures / Tables
- Optional but professional.
- Triggers off captions — useless if captions aren't in use.

---

### Bookmarks
- Used for internal hyperlinks (e.g., appendix references).
- Also used by macros to mark sections for automatic removal based on assessment type.

---

### Custom Dictionary
- Auto-corrects common misspellings and embarrassing typos (e.g., "pubic" → "public").
- Lives on the local machine — each team member must configure their own.

---

### Language Settings
- Apply "ignore spelling/grammar" to your code/terminal font style.
- Prevents the spell checker from flagging every command and tool output in your figures.
- Saves enormous time during final spell check.

---

### Custom Bullet / Numbering
- Set up auto-numbering for findings, appendices, and any sequenced content.

---

### Quick Access Toolbar — Recommended Additions
*(File → Options → Quick Access Toolbar)*

| Button | Why |
|---|---|
| **Back** | After clicking an internal hyperlink to verify it, returns you to where you were |
| **Undo / Redo** | If you don't use keyboard shortcuts |
| **Save** | If you don't use `Ctrl+S` constantly (you should) |
| Browse "Commands Not in Ribbon" | Find hidden useful functions |

---

### Useful Hotkeys

| Hotkey | Action |
|---|---|
| `F4` | Repeat last action (e.g., apply same font style to next selection) |
| `Ctrl+A` → `F9` | Update all fields (ToC, LoF, LoT) simultaneously — use carefully |
| `Ctrl+S` | Save — do it constantly |
| `Ctrl+Alt+S` | Split window into two panes (view two sections simultaneously) |
| `Shift+F5` | Jump to last edit location (useful if you accidentally type somewhere random) |

---

### Automation via Macros

When you have a mature template but no budget for a full reporting platform, macros cover a lot of ground.

**What macros can do:**
- Pop-up dialog at report creation to fill: client name, dates, scope, environment names, testing type → auto-inserts into placeholders throughout the document
- Combine multiple assessment type templates into one `.dotm` and use bookmarks to strip out irrelevant sections automatically
- Single template to maintain instead of many
- Automate common QA corrections (e.g., consistent terminology)

> Requires Windows Word + VB Editor. Mac Word's VB Editor is functionally useless.

---

## Reporting Tools & Findings Database

After a few engagements, you'll notice the same findings appearing repeatedly. Without a database, you waste time rewriting identical content and risk inconsistency across your team.

### Minimum viable approach:
A sanitised Word/Markdown document with stock findings you can copy-paste and customise per engagement.

### Dedicated platforms:

| Free | Paid |
|---|---|
| Ghostwriter | AttackForge |
| Dradis | PlexTrac |
| Security Risk Advisors VECTR | Rootshell Prism |
| WriteHat | |

> Stock findings still **must** be customised per client environment — never paste generics verbatim.

---

## Misc Reporting Tips & Tricks (Master Checklist)

### Content
- [ ] Tell a story — why does Kerberoasting matter? What was the real-world impact of those default creds?
- [ ] Write as you go — document during testing, not after
- [ ] Keep notes in chronological order
- [ ] Show enough evidence to reproduce — but don't clutter with unnecessary screenshots

### Screenshots & Evidence
- [ ] Use **Greenshot** (or equivalent) to add arrows, coloured boxes, and annotations
- [ ] Redact credentials, hashes, and secrets using solid shapes — **not blur** (blur can be reversed)
- [ ] Always show target IP (`ipconfig`/`ifconfig`) or URL bar to prove it's the client's host
- [ ] Turn off bookmarks bar and personal extensions before screenshotting browser
- [ ] Console screenshots: solid black background, white/green text, no transparent window bleeding
- [ ] Consider light background + dark text for console screenshots if client will print the report

### Tool Output
- [ ] Redact `(Pwn3d!)` from CrackMapExec output — change in CME config file so you don't have to do it every time
- [ ] Check Hashcat candidate password output — remove any crude/offensive words from wordlist results
- [ ] These are the **rare** acceptable cases for modifying tool output — only when it's offensive/unprofessional and not changing the finding's evidentiary value

### Professionalism
- [ ] Keep hostname and username professional (not `azzkicker@clientsmasher`)
- [ ] Spell check, grammar check, consistent fonts and font sizes
- [ ] Spell out acronyms on first use
- [ ] Use Grammarly or LanguageTool — but **check your MSA** first (these tools may send data to the cloud)

### Safety & Backup
- [ ] Enable autosave in notetaking tool AND Word
- [ ] Back up notes and evidence to a secondary location throughout the engagement
- [ ] Do not store everything on a single VM — VMs fail
- [ ] Automate backup wherever possible

### Process
- [ ] Script and automate repetitive tasks for consistency across assessments
- [ ] Establish and follow a style guide
- [ ] Establish a QA checklist inside your template (remove before delivery)

---

## Client Communication Protocol

### Start of Engagement — Send a Start Notification Email

Include:
- Tester name
- Type and scope of engagement
- Source IP address (public IP for external, internal IP for internal pentest)
- Anticipated testing dates
- Primary and secondary contact info (email + phone)

---

### During the Engagement
- Maintain open dialogue — don't go dark
- **Discovered additional scope?** (subnet, subdomain) → ask the client if they want to add it
- **Found critical RCE/SQLi on external asset?** → stop testing, formally notify, await instructions
- **Host went down from scanning?** → be upfront immediately, provide logs
- **Got Domain Admin?** → notify the client so they can manage alerts and prep management for the report. Ask if there are additional targets to focus on

---

### End of Each Day — Send a Stop Notification

Include:
- Signal end of testing activities (protects you if client investigates alerts)
- High-level summary of findings (especially if 20+ highs are coming — don't blindside them)
- Re-confirm expected report delivery timeline

---

### Why This Matters
- Creates a documented window of your testing activities → protects you if blamed for an outage
- Builds a trusted advisor relationship → repeat business and referrals
- If a client asks "did you hit host X on Tuesday?" → you should have documented evidence to answer definitively

---

## QA Process

### Why QA Is Non-Negotiable

> "The report is your highlight reel. It's what the client is paying for."

A sloppy report calls everything into question — including the quality of your testing.

### QA Rules

| Rule | Detail |
|---|---|
| At least **one** external reviewer | Never QA your own work if avoidable |
| Ideally **two rounds** | Technical accuracy first, then styling/cosmetics |
| Solo? | Sleep on it — return with fresh eyes before reviewing |
| Use **Track Changes** | So you can learn from corrections and stop repeating mistakes |
| Don't send a sloppy draft to QA | Fix grammar/spelling yourself first — getting it kicked back wastes everyone's time |

### What the Author Fixes (not QA):
- Missing or poorly illustrated evidence
- Missing findings
- Unusable executive summary
- Any significant content gap

### What QA Can Fix:
- Minor typos and phrasing
- Small formatting issues

### QA Checklist in Template
- Embed a QA checklist directly in your report template
- Remove before delivery
- Grows over time as your team identifies recurring mistakes

### QA Tracking at Scale
- Small team: Google Sheet to track report status
- Larger team: Jira or equivalent PM tool

---

## Report Delivery Flow

```
Testing complete
        ↓
Self-review (author)
        ↓
Internal QA Round 1 (technical accuracy)
        ↓
Internal QA Round 2 (styling + cosmetics) [optional]
        ↓
Issue DRAFT to client
        ↓
Client review period (~1 week)
        ↓
Report Review Meeting
        ↓
Incorporate client feedback (if any)
        ↓
Issue FINAL report
        ↓
Archive all testing data (per retention policy)
```

---

## Draft vs. Final

- Deliver **DRAFT** first after internal QA is complete
- After review meeting and any changes → issue **FINAL**
- Even if no changes, the FINAL designation matters — some auditors will only accept a "Final" report as a compliance artifact

---

## Report Review Meeting

- Offer after client has had ~1 week to review
- Walk through findings one by one
- Answer questions about what you found and how
- If you answer the same questions every engagement → update your template to pre-answer them
- Helps you improve your ability to present technical data to non-technical audiences

---

## Key Takeaways

- **Report as you go** — a report written live is always better than one reconstructed from memory at the end.
- **Templates + macros** are a force multiplier — invest time upfront, save hours on every future engagement.
- **A findings database** is non-negotiable once you're doing regular work — inconsistency across a team is a quality killer.
- **Client communication** isn't soft skills fluff — it's legal and professional protection and a business development tool.
- **QA is mandatory** — the report is the only thing the client actually sees of your work. Make it count.
- **Redact everything sensitive** — credentials, hashes, offensive tool output, profane wordlist candidates.
- **Draft → Final flow** matters for compliance-minded clients — never skip it.
- Online grammar tools (Grammarly, LanguageTool) may transmit data to the cloud — check your MSA before using on confidential reports.

---

## Gotchas

- Never modify a previous client's report for a new engagement — too easy to leave the old client's data behind.
- Blur is **not** acceptable for redacting sensitive info in screenshots — use solid shapes. Blur can be reversed.
- Changing CrackMapExec's `(Pwn3d!)` in the config file is better than fixing it in Word every time.
- Grammar tool cloud sync may breach your client's MSA — verify before enabling.
- Solo operators: "sleeping on it" is a real QA technique, not an excuse to skip review.
- Always use Track Changes during QA — otherwise you'll repeat the same mistakes indefinitely.
