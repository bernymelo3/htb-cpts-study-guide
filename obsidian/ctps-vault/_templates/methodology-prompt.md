# Methodology Generation Prompt (snippet)

Save this as a Raycast / Alfred / text-expansion snippet so you don't retype it. Fill in the `<MODULE-FOLDER>` placeholder before sending.

---

Create an exam-ready methodology playbook for the CPTS module at:
**`CPTS/obsidian/ctps-vault/<MODULE-FOLDER>/`**

Follow the same format as the existing reference:
`CPTS/obsidian/ctps-vault/file-uploads/00-METHODOLOGY.md`

The most recent additional reference (closer in style/tone to current work):
`CPTS/obsidian/ctps-vault/xss/00-METHODOLOGY.md`

## Steps

1. **Check the registry first:** Read `CPTS/obsidian/ctps-vault/METHODOLOGY-REGISTRY.md` to confirm this module is still pending and note the reserved ID.
2. Read every `.md` file in the target module folder to understand all techniques, commands, gotchas, and lab Q&A patterns.
3. Create `<MODULE-FOLDER>/00-METHODOLOGY.md` following the file-uploads / xss methodology structure:
   - Frontmatter (`## ID`, `## Module`, `## Kind` = `methodology`, `## Title`, `## Description`, `## Tags`)
   - **TL;DR phase flow** — 5–6 phases from recon to post-exploitation
   - **One section per phase** with concrete commands and decision logic
   - **Decision Tree** ASCII block routing from initial signal → exploit path
   - **Filter / signal → bypass reference table** mapping detection signal → counter-move
   - **Copy-paste payloads / one-liners** organized by scenario
   - **Top Gotchas** drawn from the user's own session notes — quote them, don't invent
   - **Related Vault Notes** cross-links to every section note in the folder
4. Use the reserved ID from `METHODOLOGY-REGISTRY.md` (700+ block — see the "ID Allocation" table).
5. After writing, regenerate `SEARCH.md` from the vault root using the bash one-liner at the top of `SEARCH.md` (NUL-delimited find). Verify the new methodology entry appears.
6. **Update `METHODOLOGY-REGISTRY.md`:** flip the row from `Pending` → `Done`, fill in the ID and date, and re-evaluate "Next Up".

## Constraints

- Use only techniques, commands, and gotchas that appear in the existing notes. **Do NOT invent payloads or fabricate citations.** Quote the user's own commands over textbook defaults.
- Match the tone of the file-uploads / xss methodology: terse, scannable under exam pressure, decision-tree first, prose last.
- Markdown only (no HTML), tables welcome, code blocks for commands.
- Don't touch other module folders.
- Don't commit.

## Tips

- Run on **one module per conversation** — clear between each. Each module's notes are 5–10 files of ~200 lines, so a fresh context handles it cleanly.
- For modules with **many sections** (e.g. `password-attacks` once it has notes, `ad-enum-attacks` already at 33), append to the prompt:
  > *This module is large — read files in parallel and skim aggressively.*
- If a module folder has **fewer than 3 notes**, the methodology will be thin — skip it (mark `Skip` in the registry) until more notes are added.
- After writing, the agent should ask whether to proceed to the next pending module or stop.

## Order to tackle (smallest first)

See live order in `METHODOLOGY-REGISTRY.md`. Current queue (smallest unwritten → largest):

1. `documentation/` (8) — reporting, may need different shape
2. `sqlmap-fundamentals/` (11)
3. `file-inclusion/` (12)
4. `login-brute-forcing/` (12)
5. `pivoting-tunneling/` (12)
6. `common-services/` (15)
7. `sql-injection-fundamentals/` (17)
8. `web-proxies/` (17)
9. `ad-enum-attacks/` (33) — append parallel-read note

Skip `password-attacks/` (1 note) until more notes land.
