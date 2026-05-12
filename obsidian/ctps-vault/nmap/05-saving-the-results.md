# NOTE — Saving the Results

## ID
5

## Module
Nmap

## Kind
notes

## Title
Section 5 — Saving the Results

## Description
Save scan output in normal / grepable / XML formats with `-oA`, then convert XML to a polished HTML report with `xsltproc`.

## Tags
nmap, output, reporting, xml, xsltproc, documentation

## Commands
- `sudo nmap <IP> -p- -oA target`
- `sudo nmap --open -p- <IP> -T5 -oX report.xml`
- `xsltproc target.xml -o target.html`
- `xsltproc report.xml -o report.html`
- `firefox report.html`

## Output Formats
| Flag | Extension | Purpose |
|------|-----------|---------|
| `-oN` | `.nmap` | Normal — same as terminal output. |
| `-oG` | `.gnmap` | Grepable — one host per line, easy `grep`/`awk`. |
| `-oX` | `.xml` | XML — machine-readable, inputs into other tools, convertible to HTML. |
| `-oA <basename>` | all three | Saves as `<basename>.{nmap,gnmap,xml}` in one shot. |

## Why Always `-oA`
- You can diff scans across time / from different positions in the network.
- HTML reports impress non-technical stakeholders during reporting.
- Ingestion: tools like Eyewitness, Metasploit, Faraday consume Nmap XML.

## XML → HTML Report
```bash
sudo nmap --open -p- <IP> -T5 -oX report.xml
xsltproc report.xml -o report.html
firefox report.html
```
`xsltproc` uses Nmap's bundled stylesheet. The result is a clean, sortable, table-formatted HTML page — useful in a deliverable.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Highest open port from full TCP scan | **31337** | Full port scan → 7 open ports, highest is 31337 (Elite). Optional: convert XML to HTML for the report |

### Reference commands
```bash
sudo nmap --open -p- <IP> -T5 -oX report.xml
xsltproc report.xml -o report.html
firefox report.html
```

## Key Takeaways
- Use `-oA <basename>` reflexively. It costs nothing and saves you from rerunning slow scans.
- XML output is the canonical format — keep it as the source of truth and generate other formats from it.
- HTML reports via `xsltproc` are exam/report-ready out of the box.

## Gotchas
- Without a full path, Nmap saves output to the current working directory — easy to lose if you `cd` afterwards.
- Re-running with the same `-oA <name>` overwrites previous output silently. Use timestamped names for long engagements.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|nmap]]
← [[04-host-and-port-scanning]] | [[06-service-enumeration]] →
<!-- AUTO-LINKS-END -->
