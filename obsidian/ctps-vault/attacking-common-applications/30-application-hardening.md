## ID
707

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 30 — Application Hardening

## Description
Theory section covering general and application-specific hardening guidelines for all applications discussed in the module, from authentication and access controls to monitoring and inventory management.

## Tags
hardening, defense, best-practices, access-controls, authentication, theory

## What This Section Covers
This closing section shifts to the defensive perspective, outlining how organizations should harden the applications covered throughout the module. It emphasizes creating accurate application inventories, applying universal hardening principles (strong auth, access controls, updates, monitoring), and taking application-specific measures. Understanding defense helps pentesters identify what's missing during assessments.

## General Hardening Principles
1. **Application inventory** — Maintain an accurate list of all internal and external-facing apps; use tools like Nmap and EyeWitness to discover shadow IT, deprecated installs, and unlicensed tools (e.g., Splunk free edition losing auth)
2. **Secure authentication** — Enforce strong passwords, change default admin credentials, disable default admin accounts and create custom ones, mandate 2FA for admin-level users
3. **Access controls** — Login pages shouldn't be externally accessible without business justification; configure file/folder permissions to deny unauthorized uploads or deployments
4. **Disable unsafe features** — E.g., disable PHP code editing in WordPress to prevent code execution post-compromise
5. **Regular updates** — Patch promptly; apply vendor-supplied security patches as soon as available
6. **Backups** — Configure website and database backups to a secondary location for quick recovery
7. **Security monitoring** — Use WAFs, security plugins, and monitoring tools as an additional defense layer (not a silver bullet)
8. **LDAP/AD integration** — Single sign-on reduces credential sprawl, improves auditing (especially with Azure sync), and enables fine-grained password policies

## Application-Specific Hardening
- **WordPress** — Use security plugins like WordFence (monitoring, suspicious activity blocking, country blocking, 2FA)
- **Joomla** — Use AdminExile plugin to require a secret key for admin login (e.g., `/administrator?thisismysecretkey`)
- **Drupal** — Disable, hide, or move the admin login page
- **Tomcat** — Limit Manager and Host-Manager to localhost only; if external access required, enforce IP whitelisting + strong non-default creds
- **Jenkins** — Configure permissions using the Matrix Authorization Strategy plugin
- **Splunk** — Change default password, ensure proper licensing to enforce authentication
- **PRTG** — Stay updated, change default PRTG password
- **osTicket** — Limit internet access if possible
- **GitLab** — Enforce sign-up restrictions: require admin approval for new sign-ups, configure allowed/denied domains

## Key Takeaways
- An accurate application inventory is the foundation — you can't protect what you don't know exists
- Shadow IT, deprecated apps, and trial-to-free conversions (like Splunk losing auth) are common findings during pentests
- Default credentials remain one of the most impactful findings — Tomcat Manager, PRTG, Nagios, printers with LDAP creds
- From the offensive perspective, understanding hardening helps identify what controls are missing: no 2FA? Default admin accounts? Manager accessible from the internet?
- Regular assessments should verify that remediation recommendations from pentest reports have actually been implemented
- Ask the question: does this app really need to be externally accessible? Reducing attack surface is the highest-impact control
