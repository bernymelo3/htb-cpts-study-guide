# Section 2 — Application Discovery & Enumeration

## ID
530

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 2 — Application Discovery & Enumeration

## Description
Covers web application discovery using Nmap, EyeWitness, and Aquatone to efficiently screenshot and categorize web services across large scopes.

## Tags
enumeration, eyewitness, aquatone, nmap, web-discovery, recon

## Commands
- `nmap -p 80,443,8000,8080,8180,8888,10000 --open -oA web_discovery -iL <SCOPE_LIST>`
- `sudo nmap --open -sV <TARGET_IP>`
- `sudo apt install eyewitness`
- `eyewitness --web -x <NMAP_XML> -d <OUTPUT_DIR>`
- `eyewitness --web -f <URL_LIST> -d <OUTPUT_DIR>`
- `cat <NMAP_XML> | aquatone -nmap`
- `cat <NMAP_XML> | aquatone -nmap -out <OUTPUT_DIR>`

## What This Section Covers
How to efficiently discover and inventory web applications across a target scope using Nmap for port scanning, then EyeWitness and Aquatone to automatically screenshot and categorize every web service. This is critical for large environments where manually browsing hundreds of hosts is impractical. The section also covers organizing enumeration data and setting up a notetaking structure for engagements.

## Methodology

1. **Create a scope list** with all in-scope hostnames and IPs, add vHost entries to `/etc/hosts`:
   ```
   cat << EOF > scopeList
   app.inlanefreight.local
   dev.inlanefreight.local
   drupal-dev.inlanefreight.local
   drupal-qa.inlanefreight.local
   drupal-acc.inlanefreight.local
   drupal.inlanefreight.local
   blog.inlanefreight.local
   EOF
   ```

2. **Run Nmap against common web ports** using the scope list, saving output in all formats (`-oA`):
   ```
   sudo nmap <TARGET_IP> -p 80,443,8000,8080,8180,8888,10000 --open -oA webDiscovery -iL scopeList
   ```

3. **Feed Nmap XML into EyeWitness** to generate a screenshot report:
   ```
   eyewitness --web -x webDiscovery.xml -d inlanefreight_eyewitness
   ```

4. **Feed Nmap XML into Aquatone** for a second screenshot report:
   ```
   cat webDiscovery.xml | aquatone -nmap
   ```

5. **Review reports** — prioritize High Value Targets (Tomcat, Jenkins, Splunk, GitLab, etc.), note down dev hosts, default credential candidates, and anything unexpected.

6. **Run deeper scans** — while reviewing initial results, launch a broader Nmap scan (top 10,000 or all TCP ports) and re-run screenshotting tools against the new results.

7. **Perform service version scans** (`-sV`) on interesting hosts for more detail:
   ```
   sudo nmap --open -sV <TARGET_IP>
   ```

## Key Observations from the Example Environment

- **10.129.201.50 (Windows)**: IIS on 80, Splunk on 8000/8089, PRTG on 8080, RDP on 3389
- **app-dev.inlanefreight.local**: Multiple web ports (80, 8000, 8080, 8180, 8888) — heavy app server, likely Tomcat
- **gitlab-dev.inlanefreight.local**: Git repos may contain creds or clues; check if self-registration is open
- **Hosts with `dev` in FQDN**: Higher chance of debug mode, untested features, weaker security

## EyeWitness Details

- Accepts Nmap XML (`-x`), Nessus XML (`-x`), or URL list (`-f`)
- Uses Selenium to render and screenshot pages
- Auto-categorizes apps (High Value Targets, CMS, 401/403, Splash Pages)
- Fingerprints apps and suggests default credentials
- Creates: `ew.db`, `report.html`, `screens/`, `source/`, `open_ports.csv`, `Requests.csv`
- Install: `sudo apt install eyewitness` or clone repo and run `setup.sh`

## Aquatone Details

- Accepts Nmap XML via pipe with `-nmap` flag, or a `.txt` host list
- Also uses headless Chrome/Chromium for screenshots
- Clusters pages by similarity — useful for spotting patterns
- Output: `aquatone_report.html` plus session/screenshot files in current directory
- Title page header says **"Pages by Similarity"**

## Report Review Priorities

- **Tomcat**: Try default creds on `/manager` and `/host-manager` → WAR upload → RCE via JSP
- **Jenkins**: Check for unauthenticated Script Console access
- **Splunk**: Free/trial license may convert to no-auth version
- **GitLab**: Check for open registration, public repos, credential leaks in commit history
- **osTicket / support portals**: May leak sensitive info or allow email domain registration for social engineering
- **Custom apps**: Always worth testing — wide variety of vulns possible
- **Anything unexpected**: File upload pages, directory listing, dev/debug endpoints

## Notetaking Structure for Engagements

```
External Penetration Test - <Client Name>
├── Scope (IPs, URLs, fragile hosts, timeframes, limitations)
├── Client Points of Contact
├── Credentials
├── Discovery/Enumeration
│   ├── Scans (Nmap, Nessus, Masscan — timestamped with exact syntax)
│   ├── Live Hosts
│   └── Application Discovery
│       ├── Scans (EyeWitness, Aquatone reports)
│       └── Interesting/Notable Hosts
├── Exploitation
│   ├── <Hostname or IP>
│   └── <Hostname or IP>
└── Post-Exploitation
    ├── <Hostname or IP>
    └── <Hostname or IP>
```

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Name of .db file EyeWitness creates | ew.db | `ls inlanefreight_eyewitness/` after running EyeWitness |
| Q2: Header on aquatone_report.html title page | Pages by Similarity | Open `aquatone_report.html` in browser |

## Key Takeaways

- Always save Nmap output in all formats (`-oA`) — the XML is input for EyeWitness/Aquatone
- Enumeration is iterative: initial common-port scan → screenshot → deeper scan → screenshot again
- Don't start attacking during discovery phase — finish the full report review first to avoid tunnel vision
- `dev` hosts and forgotten/demo applications are often the easiest wins
- Timestamp every scan and record exact syntax — clients may ask about observed activity
- Set up your report skeleton at the start of the engagement, not the end
- Internal pentests surface even more: printer pages (cleartext LDAP creds), ESXi/vCenter, iLO/iDRAC, IoT devices

## Gotchas

- EyeWitness and Aquatone both need a headless browser (Selenium/Chrome/Chromium) — install issues are common on fresh VMs
- Aquatone warns about "unreliable Google Chrome" — install Chromium for better results
- Not all hosts in the module's scope list are accessible when spawning the target; the exercise uses a subset
- Some applications only appear on non-standard ports — the initial common-port scan is a starting point, not comprehensive
