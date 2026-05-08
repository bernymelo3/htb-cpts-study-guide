## ID
114

## Module
Attacking Common Services

## Kind
notes

## Title
Section 14 — Latest DNS Vulnerabilities

## Description
Subdomain takeover: when a DNS CNAME record points to a deactivated third‑party service (AWS S3, GitHub, etc.), an attacker can register that resource and gain full control over the subdomain – enabling phishing, cookie theft, and other attacks.

## Tags
theory, dns, subdomain-takeover, cname, misconfiguration, bug-bounty

## TL;DR — What's Important
- **Subdomain takeover** occurs when a CNAME record points to a third‑party resource (e.g., `bucket.s3.amazonaws.com`) that no longer exists. The attacker registers the expired resource and controls the subdomain.
- **Root cause:** Companies forget to remove DNS records after cancelling a service. DNS entries cost nothing, so they linger.
- **Impact:** Phishing campaigns using the company’s legitimate domain, stealing cookies (session hijacking), CSRF, bypassing CSP, abusing CORS – all from a trusted domain.
- **Detection:** Look for CNAME records pointing to external services. Check if those services respond with errors like `NoSuchBucket`, `404 - There isn't a GitHub Pages site here`, or `NoSuchHost`.
- **Attack cycle:** Source = outdated subdomain name → Process = DNS resolution → Privileges = domain admin (implicit trust) → Destination = attacker’s server.

## Concept Overview
Subdomain takeover is a common misconfiguration in DNS management. When an organization uses third‑party services (AWS S3, GitHub Pages, Azure, Heroku, etc.), they create a CNAME record that points from a subdomain (e.g., `assets.company.com`) to the service’s endpoint (e.g., `company-assets.s3.amazonaws.com`). If the company cancels the service but does not remove the DNS record, the CNAME remains. An attacker can then register the expired resource (e.g., recreate the S3 bucket) and gain full control over the subdomain. Visitors to the subdomain see the attacker’s content, fully trusting it because it resides under the company’s legitimate domain. This has led to significant bug bounty payouts (HackerOne has a dedicated category) and real‑world phishing campaigns.

## Key Concepts

### CNAME Record
- Canonical Name record – maps a domain alias to another domain (the canonical name).
- Example: `support.company.com` CNAME `company.s3.amazonaws.com`
- The browser resolves `support.company.com` to the IP of `company.s3.amazonaws.com`.

### How Subdomain Takeover Happens
1. Company creates subdomain `project.company.com` pointing to a third‑party service (AWS S3, GitHub Pages, etc.).
2. Company later stops using that service but **forgets to delete the CNAME record**.
3. The third‑party resource becomes available for anyone to register (e.g., the S3 bucket name is free).
4. Attacker registers the resource (creates the bucket, GitHub Pages repo, etc.).
5. Now `project.company.com` resolves to the attacker’s content – a full takeover.

### Vulnerable Third‑Party Services (Common Examples)
| Service | CNAME Example | Error Indicator |
|---------|---------------|------------------|
| AWS S3 | `bucket.s3.amazonaws.com` | `NoSuchBucket` |
| GitHub Pages | `username.github.io` | `404 - There isn't a GitHub Pages site here` |
| Heroku | `app.herokuapp.com` | `No such app` |
| Azure | `cloudapp.net` | `404 Not Found` |
| Fastly | `global.prod.fastly.net` | `Fastly error: unknown domain` |
| Shopify | `myshopify.com` | `Sorry, this store is unavailable` |
| WordPress VIP | `vip.wordpress.com` | `No site found` |

### Attack Cycle (as described in the module)

**First cycle – registering the takeover:**

| Step | Action | Concept Category |
|------|--------|------------------|
| 1 | Attacker discovers an unused subdomain via CNAME enumeration | Source |
| 2 | Attacker registers the expired third‑party resource | Process |
| 3 | The domain owner’s DNS entry (CNAME) remains, granting trust | Privileges |
| 4 | The resource is hosted on the attacker’s server (Destination) | Destination |

**Second cycle – victim interaction:**

| Step | Action | Concept Category |
|------|--------|------------------|
| 5 | Victim visits the subdomain (e.g., `project.company.com`) | Source |
| 6 | DNS resolves CNAME to attacker‑controlled resource | Process |
| 7 | Browser trusts the domain because it’s under `company.com` | Privileges |
| 8 | Victim is forwarded to the attacker’s content | Destination (network) |

## Why It Matters
- **Trust is the vulnerability.** Users and browsers trust `*.company.com`. A subdomain takeover gives the attacker a foothold in that trusted namespace.
- **Phishing** – Attackers can clone login pages and harvest credentials. Victims see `login.company.com` in the address bar – no suspicion.
- **Session hijacking** – If the subdomain can set cookies (depending on domain flags), an attacker might steal or overwrite cookies for the parent domain.
- **Bypassing CSP / CORS** – A taken‑over subdomain can often execute scripts on the main domain if security policies are misconfigured.
- **Bug bounties** – Subdomain takeovers are a well‑paid category because the impact is high and the fix is simple (remove the CNAME). HackerOne has paid tens of thousands of dollars for such reports.

## Defender Perspective
- **Detection:** Regularly audit DNS records. Look for CNAMEs pointing to external services. Use tools like `subfinder` + `nuclei` (with takeover templates) or `tko-subs`, `can-i-take-over-xyz`.
- **Mitigation:**  
  - Before deleting a third‑party service, **remove the CNAME record** first.  
  - Use a DNS monitoring solution that alerts on stale CNAME records.  
  - Implement a “DNS hygiene” policy – quarterly reviews of all external‑pointing records.  
  - Use cloud‑native tools (AWS Config, Azure Policy) to detect orphaned resources.
- **MITRE ATT&CK:** T1584 (Compromise Infrastructure – registering expired domains/resources), T1566 (Phishing), T1190 (Exploit Public‑Facing Application – if the taken‑over subdomain hosts a malicious app).

## Key Takeaways
- Subdomain takeover is **not a software vulnerability** – it’s a configuration error. The DNS server functions correctly; the company simply left a pointer to a dead endpoint.
- The **CNAME record itself is the vulnerability**. If the target resource no longer exists, an attacker can claim it.
- Always check CNAMEs during DNS enumeration. Tools like `host -t CNAME subdomain.domain` or `dig CNAME` can reveal them.
- Proof of concept (PoC) for a takeover is simple: register the resource and show that you can control the subdomain (e.g., serve a custom HTML page). Many bug bounty programs accept this.
- A subdomain takeover can escalate to full account compromise if the main domain accepts authentication tokens from the subdomain without proper validation.

## Gotchas
- Not all CNAMEs are vulnerable. Some services (e.g., CloudFront) have domain‑specific validation that prevents arbitrary registration.
- The takeover only works if the **entire CNAME target** can be registered. For example, `sub.company.com` → `user.github.io` – you need to control the GitHub account that owns `user.github.io`.
- Some services have expiration windows. A bucket may be unavailable for reuse immediately after deletion. Check the service’s policy.
- Subdomain takeover does not give you control over the parent domain – only that specific subdomain. However, that may be enough for phishing or session attacks.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|common-services]]  
← [[13-attacking-dns]] | [[15-attacking-email-services]] →
<!-- AUTO-LINKS-END -->
