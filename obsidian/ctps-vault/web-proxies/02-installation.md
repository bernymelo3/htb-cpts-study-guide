- **First‑run project selection:**
- **Community Edition:** Only *Temporary Project* is available — you cannot save and reload.
- **Professional Edition:** You can create a named, persistent project.
- **Configuration:** Choose *Use Burp Defaults* for now; custom configurations can be loaded later.

### OWASP ZAP Installation & Launch
- Download from the [ZAP download page](https://www.zaproxy.org/download/).
- Install the appropriate package.
- Launch via the application menu or with the `zaproxy` command.
- **Session persistence:** When the window appears, you can:
- Persist the session with a timestamped name (useful for multi‑day tests).
- Specify a custom name and location.
- Choose *No, I do not want to persist this session at this moment in time* for a temporary project.
- Unlike Burp Community, ZAP’s free version allows persistent sessions by default.

### Quick‑start Tips
- Both tools use Java; the required JRE is normally bundled with the installer.
- Dark theme: Burp → *Settings → User interface → Display → Theme: Dark*; ZAP → *Tools → Options → Display → Look and Feel: Flat Dark*.

## Why It Matters
Knowing how to install and launch your proxy on any platform ensures you are never stuck during an engagement. The project‑type decision (temporary vs persistent) matters when you need to save results for a report or resume a large assessment later. Starting with correct defaults avoids misconfigurations that could break the proxy later.

## Key Takeaways
- Burp Community’s temporary‑only limitation is a deliberate restriction; if you need to save, consider ZAP or Burp Pro.
- The `java -jar` method works on any OS with Java and is handy when you don’t want to install (or need a portable version).
- Persisting a ZAP session automatically creates a file with all your scanned history, alerts, and configurations — good for reporting.
- Always check you’re running the latest stable version; both tools update frequently to handle new TLS and web features.

## Gotchas
- If you download the JAR but lack a Java Runtime, the tool will not start. Ensure JRE (Java 17+ for recent Burp versions) is installed.
- On macOS, Burp may request accessibility permissions to control the browser; grant them or the embedded browser won’t work.
- Choosing *No* to persistence in ZAP means all session data is lost when you close the application, so be sure before you spend hours scanning.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-proxies]]  
← [[01-intro]] | [[03-proxy-setup]] →
<!-- AUTO-LINKS-END -->
