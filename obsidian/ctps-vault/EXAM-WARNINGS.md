# CPTS Exam — What Can Go Wrong

A pre-exam pitfall checklist tailored to **this vault's actual state**. Read this 24h before starting the exam, then again on Day 1 morning.

**Exam window:** 10 days (10) — 7 days for hacking + buffer, **at least 1.5 full days reserved for the report**. The report is the part that fails people.

---

## 0. Vault gaps you must be aware of going in

These are the modules where you currently have **no notes** or thin notes. Before exam start, either fill them or accept that you'll be working from HTB Academy / external sources for that topic. Do not discover this mid-exam.

| Topic | Vault state | Risk level |
|---|---|---|
| `nmap/` | empty | **HIGH** — recon is step 1 of every box |
| `footprinting/` | empty | **HIGH** — service-by-service enumeration is the core skill |
| `ffuf/` | empty | **HIGH** — directory/vhost/parameter fuzzing on every web target |
| `web-recon/` | empty | **HIGH** — info gathering web edition (subdomains, vhosts, OSINT) |
| `shells-payloads/` | empty | **HIGH** — revshells, msfvenom, payload formats — needed every box |
| `linux-privesc/` | empty | **HIGH** — at least one Linux box almost certain |
| `windows-privesc/` | empty | **HIGH** — at least one Windows box almost certain |
| `password-attacks/` | overview stub only | **MEDIUM** — the techniques are listed but per-tool detail missing (john, hashcat, SAM dump, LSASS, DCSync, PtH, PtT, PtC) |
| `ad-enum-attacks/` §26, §28, §30, §33, §34, §35 | missing | **MEDIUM** — tail of trust attacks + hardening; main chain is covered |

**Decision before exam start:** for each HIGH-risk row, decide *now* whether you'll (a) write the notes, (b) bookmark the HTB module pages, or (c) prepare a one-page cheat sheet. Don't enter the exam with this unresolved.

---

## 1. Recon — the #1 reason people get stuck

Pitfalls (advice paraphrased from a real CPTS pass write-up; emphasized because the vault is weak here):

- **Skipping or rushing recon.** "Most people get stuck because they didn't enumerate properly." If you can't move forward, you missed a port, service, share, vhost, subdomain, or parameter — go back, do not invent exploits.
- **Not re-running recon on every new host.** Every shell, every pivot target — `nmap`, banner-grab, check shares, check for creds in config / bash history. Don't assume the next box is similar to the last.
- **Forgetting the "soft" recon surfaces:**
  - Zone transfer (`dig axfr`)
  - vhost fuzzing (Host-header) — different from subdomain fuzzing
  - Parameter fuzzing (`ffuf -w params.txt -u 'host/page.php?FUZZ=test'`)
  - SMB null sessions (`smbclient -L`, `enum4linux-ng`)
  - LDAP anonymous bind
  - SNMP public community
- **Closed ports lying.** UDP scans, full TCP scans (`-p-`), and slower scans find what `-T4 --top-ports 1000` misses. Run a full TCP scan in the background while you start exploiting the easy stuff.
- **Not reading service banners carefully.** Outdated software → known CVE. The version is in the banner.

**Vault hooks:** none — you have to write these or rely on HTB notes. Write at minimum a one-page cheat: nmap flags, ffuf flags, common service-enum commands. Mirror the structure used in `common-services/` and `pivoting-tunneling/`.

---

## 2. Exploitation phase — common foot-guns

- **Tool monoculture.** Use both auto and manual. `sqlmap` first, then manual SQLi if it stalls. `nuclei` / `nikto` to fingerprint, then exploit by hand.
- **Burning hours on the wrong vuln class.** Set a soft 60–90 min timer per technique. If LFI isn't yielding RCE in that window, switch to a different angle (login brute, IDOR, parameter fuzzing). `ATTACK-PATHS.md` is built for this — use it.
- **Not pivoting tooling.** Once you have *any* foothold, your next 30 min should be: dump bash history, check `/etc/`, check home dirs, check running services, look for creds in plain text. Same on Windows: PowerShell history, `C:\Users\*\Documents`, registry, scheduled tasks.
- **Reverse shell instability.** Always upgrade the shell first (`python3 -c 'import pty; pty.spawn("/bin/bash")'` + `stty raw -echo` + `export TERM=xterm`). Drops mid-pivot will cost you 30 minutes of re-exploitation.
- **Running exploits without reading them.** Public PoCs sometimes have `rm -rf /` or call back to attacker-controlled domains. Read before executing on the exam target.

**Vault hooks:** `ATTACK-PATHS.md` (state → next note), `SEARCH.md` (keyword → note), per-module `00-METHODOLOGY.md` files.

---

## 3. Active Directory — where chains break

You have strong AD coverage (32 notes), but watch for:

- **Skipping internal recon when you already have creds.** Even with valid creds, run BloodHound, check `gMSA`, check ACLs, check Kerberoastable + AS-REP-roastable accounts, check delegations. Many chains hide in attributes you didn't query.
- **Not enumerating security controls.** AppLocker / Defender / LAPS / WDAC change which payloads work. Run the `13-enumerating-security-controls.md` checks before dropping a binary.
- **Double-hop fail on WinRM.** If you compromise a user, WinRM into Host-A, then try to reach Host-B — Kerberos refuses. Note `24-winrm-double-hop-kerberos.md` exists for this exact issue.
- **Forgetting to check trusts.** If you've been in the same domain for hours and stuck, run `Get-DomainTrust` / `nltest`. The exam may require parent-trust or cross-forest pivot — `27-domain-trusts-primer.md`.
- **Cracking Kerberoast hashes you can't actually crack.** Strong service account passwords won't fall to `rockyou.txt`. Don't sink 2 hours into a hopeless hash — pivot to ACL abuse or another vector.

**Vault hooks:** `AD-PRESENTATION.md`, `AD-SIMPLE-GUIDE.md`, `ad-enum-attacks/`, `ATTACK-PATHS.md` §4.

---

## 4. Pivoting / Tunneling — where time evaporates

- **Not setting up a SOCKS proxy early.** `ssh -D 1080` + proxychains is the default; configure it the moment you have any Linux foothold inside the network.
- **proxychains-ng vs proxychains.** Different config locations (`/etc/proxychains4.conf` vs `/etc/proxychains.conf`). Verify which one your distro uses before debugging "why is this slow."
- **Tools that don't honor SOCKS.** `nmap` over proxychains is *slow and unreliable* — prefer `-sT -Pn -n` and limit ports. For RDP, use `xfreerdp` over a `chisel` TCP tunnel, not proxychains.
- **Forgetting to log tunnel commands.** When the tunnel dies you'll need to re-run the exact command for the report. Keep a `tunnels.txt` open.

**Vault hooks:** `pivoting-tunneling/00-overview.md` and §5 of `ATTACK-PATHS.md`.

---

## 5. Privilege escalation — the "I'm stuck" trap

You have **no privesc notes**. Mitigations before exam start:

- **Linux:** keep `linpeas.sh` and `pspy64` ready. Memorize: SUID hunt, sudo `-l`, cron jobs, writable `/etc/`, capabilities, kernel exploits last.
- **Windows:** keep `winPEASx64.exe`, `PowerUp.ps1`, `SharpUp`, `Seatbelt` ready. Memorize: token privileges (`SeImpersonate` → Potato), unquoted service paths, AlwaysInstallElevated, stored creds in registry/files.
- **Don't kernel-exploit first.** It's the noisiest, most fragile, and graders dislike chains that depend on a CVE when a misconfig was sitting right there.

---

## 6. Reporting — where most people fail the exam

This is the part the write-up insists on, and your `documentation/` module already covers it well. Concrete operational rules:

- **Take report-grade notes WHILE hacking, not after.** Format: "*The tester ran `<exact command>` and received the following output:*" + screenshot. Do this in real time. Re-running exploits later is the worst time-sink.
- **Screenshots immediately.** Take more than you need. Hide credentials with an opaque rectangle (use the macOS Markup tool's shape — green/black solid rectangle, not just blur).
- **Use third person, past tense, throughout the walkthrough.** "The tester used BloodHound" — not "I used BloodHound."
- **Reference each tool / exploit on first mention only.** Bloodhound → cite once. CVE-XXXX → cite once. After that, just name it.
- **Do NOT colorize usernames/groups/etc.** Keep the report monochrome and clean.
- **Use SysReptor (online) with the HTB template.** Don't try to build the report in Word.
- **Findings workflow:**
  1. Write the full walkthrough first (chronological, third person).
  2. Paste walkthrough into an AI; ask for "all findings, short summary each."
  3. For each finding, ask AI for: CVSS 3.1 vector + score, description, impact, remediation. Pick CWE manually.
  4. Evidence = copy the relevant chunk from the walkthrough (find/replace "the tester" → "the malicious actor" if desired).
- **Executive summary:** paste HTB Academy doc-and-reporting "writing a strong executive summary" → "anatomy of executive summary" + your walkthrough + finding summaries into Claude/AI; ask for an exec summary that follows those guides. Use the recommendations it generates for the recommendations section (short, not full long-form).
- **Report length is not a goal.** A clean 50-page report with crisp findings beats a padded 110-page one. The 110-page write-up was a one-day rush — not a target.
- **Time-box: do not start the report with less than 36h left.** With less than 24h you will *finish*, but you will be miserable and at risk.
- **Submission rules:** PDF or ZIP, **no password**, **≤ 20 MB**, English only. Compress screenshots if needed. Submitting ends your attempt — do not click submit until you've reviewed the rendered PDF.

**Vault hooks:** entire `documentation/` folder, especially `04-components-of-a-report.md` and `05-how-to-write-up-a-finding.md`.

---

## 7. Time / logistics — the boring stuff that breaks people

- **Pwnbox dies after 4 days, no extension.** If you're using Pwnbox, plan to spawn a fresh one OR switch to local + VPN file before day 4. Anything you've stored only on the Pwnbox is gone when it terminates — exfil notes/loot to your local machine continuously.
- **Exam VPN: Pwnbox OR VPN file, not both at once.** Switching mid-exam is fine; running both simultaneously will cause routing chaos.
- **Box instances drop below 100 min → extend immediately.** Don't let it expire mid-exploit.
- **You can reset boxes.** If you've made the box weird (failed exploit left artifacts, services crashed), reset is cheaper than debugging — but you lose any in-memory state, so document everything before resetting.
- **Sleep is a tool.** The 24-hour-awake report grind in the write-up is a warning, not a template. Plan to sleep ≥ 6h on report-eve.
- **You must submit a report even if you didn't pass.** No report = no second attempt. Always submit something.
- **Second attempt:** 14 days from feedback to start it; missing that window = loss of attempt.
- **Results take up to 20 business days.** Don't refresh email obsessively.

---

## 8. Things to prepare before clicking "Start Exam"

- [ ] Linpeas / winpeas / pspy64 / Seatbelt / SharpUp downloaded and on a USB / local folder
- [ ] Burp / ZAP licensed and configured, browser proxy ready
- [ ] BloodHound + Neo4j installed and tested (run a sample query)
- [ ] `impacket-*` suite installed (test `secretsdump.py`, `psexec.py`, `wmiexec.py`)
- [ ] `crackmapexec` / `nxc` installed and updated
- [ ] `sqlmap`, `ffuf`, `gobuster`, `feroxbuster`, `wfuzz`, `nuclei` installed
- [ ] `responder`, `mitm6`, `ntlmrelayx.py` installed
- [ ] `evil-winrm`, `xfreerdp`, `rdesktop` installed
- [ ] `john`, `hashcat`, latest `rockyou.txt`, `seclists` cloned
- [ ] `chisel`, `ligolo-ng`, `sshuttle`, `proxychains` configured
- [ ] Revshell cheat sheet (cyberchef / revshells.com) bookmarked
- [ ] SysReptor account ready, HTB CPTS template imported
- [ ] Note tool open with template per-host (IP, ports, services, creds, exploits, screenshots folder)
- [ ] Local screenshots folder per host, named `host-<ip>/01-nmap.png`, etc.
- [ ] Snippets file with your favorite one-liners (TTY upgrade, base64 transfer, SMB upload, etc.)

---

## 9. The "I am stuck" protocol (use this when frustrated)

In order, no skipping:

1. Re-read the last 30 minutes of your own notes — is there a thread you dropped?
2. Open `ATTACK-PATHS.md` and search by your current state.
3. Run a full nmap (`-p- -sV --min-rate 1000`) against the current target if you only ran top-1000.
4. Enumerate one new surface you haven't checked: vhosts, subdomains, UDP, SNMP, NFS, LDAP, IPv6.
5. Look at the box from the LAST one you compromised — did you miss creds in `~/.bash_history`, `~/.config/`, `/etc/`, mounted shares?
6. Step away for 10 minutes. Eat. Walk. Reset.
7. If still stuck after all of the above, dump the entire situation (ports, services, creds, shells, what you've tried) into Claude/AI and ask "what enumeration step have I plausibly skipped?"

Do **not** start writing custom exploits unless you're in the last 24h of hacking and have ruled out everything else. Custom exploit dev is the exam's quicksand.

---

## 10. Personal hard rules

- Never work past 1 AM on hacking. Bugs you create at 2 AM cost 2h to find at 9 AM.
- Commit notes to git every couple hours so you cannot lose them.
- After every shell, immediately dump: hostname, whoami, ip, OS version, users, groups, shares, processes — into a per-host text file. This is the report's raw material.
- If you find creds, write them in a single `creds.txt` with the format: `protocol://user:pass@host:port (source: <where found>)`. This becomes finding evidence directly.
- The exam grades the **chain**, not the boxes. Document how each step led to the next.
