# CPTS Knowledge Base

Personal study notes for the **Hack The Box Certified Penetration Testing Specialist (CPTS)** certification, organized into a searchable single-page web interface.

## Quick Start

```
open index.html
```

That's it — no server, no dependencies. Just a browser.

## What's Inside

**24 notes** across 6 categories, sourced from Notion exports and hand-written lab walkthroughs:

| Category | Notes | Covers |
|---|---|---|
| **Scanning & Enumeration** | 12 | Nmap (full port, host discovery, vuln scripts, SMTP), TTL fingerprinting, DNS/OS guessing, tcpdump, netcat |
| **Footprinting** | 3 | IMAP/POP3 protocols, DNS workflow (full `dig` cheatsheet), Remote Management (RDP, WinRM, SSH, rsync) |
| **Labs & Walkthroughs** | 6 | SMB enumeration box, DNS zone transfers, NFS enum, source port evasion (hard lab), SMTP footprinting, Medium lab full chain (NFS→SMB→RDP→MSSQL) |
| **Exploitation & AD** | 1 | Responder LLMNR poisoning |
| **Privilege Escalation** | 1 | SUID binary enumeration |
| **Tools & Techniques** | 1 | Python TTY shell upgrade |

## Features

- **Search** — filter notes by title, command, description, or tag (`Cmd+K` to focus)
- **Tag filtering** — click any tag to narrow results
- **Copy buttons** — hover any command block and click to copy
- **Prev/Next navigation** — arrow keys or buttons to step through notes
- **Related notes** — auto-suggested at the bottom of each note based on shared tags
- **Light/Dark theme** — toggle button in the bottom-left corner
- **Export** — download all notes as JSON or Markdown (bottom-right buttons)

## Keyboard Shortcuts

| Key | Action |
|---|---|
| `Cmd+K` / `Ctrl+K` | Focus search |
| `Escape` | Clear search, go home |
| `←` / `→` | Previous / Next note |

## Project Structure

```
.
├── index.html                          # The web interface (self-contained)
├── Notas Labs/
│   └── Footprinting-Medium-lab.md      # Hand-written lab walkthrough
└── ExportBlock-.../
    ├── CPTS Knowledge Base/            # Individual Notion page exports (.md)
    └── CPTS Knowledge Base *.csv       # Notion database exports
```

The original Notion exports and lab notes are kept as-is for reference. All content is embedded in `index.html` for the web interface.

## Adding New Notes

Edit the `NOTES` array inside `index.html`. Each note follows this structure:

```js
{
  id: 25,                              // unique number
  title: "Note Title",
  category: "Scanning & Enumeration",  // existing category or new one
  tags: ["nmap", "enumeration"],
  commands: ["nmap -sV <target>"],     // array of commands
  description: "What this does.",      // shown as info block
  body: ""                             // extra HTML content (optional)
}
```

## License

Personal study material. Not affiliated with Hack The Box.
