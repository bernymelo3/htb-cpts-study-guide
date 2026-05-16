# Section 16 — osTicket

## ID
907

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 16 — osTicket

## Description
Covers discovering and accessing osTicket support portal instances, logging in with obtained credentials, and extracting sensitive information (such as passwords) from closed support tickets.

## Tags
osticket, helpdesk, ticketing, credential-harvesting, information-disclosure

## Commands
- `echo '<TARGET_IP> support.inlanefreight.local' | sudo tee -a /etc/hosts`
- `curl -s http://support.inlanefreight.local/scp/login.php`

## What This Section Covers
osTicket is an open-source support ticket system commonly found in enterprise environments. From a pentester's perspective, it's valuable not because of a code-level exploit, but because **support tickets frequently contain sensitive information** — passwords, internal URLs, configuration details, PII — sent in plaintext between staff and customers. If you gain access to the staff control panel (SCP), you can mine closed tickets for credentials and other intel that advances your engagement.

## Methodology

### Phase 1 — Discovery

1. **Add vHost to `/etc/hosts`:**
   ```
   echo '<TARGET_IP> support.inlanefreight.local' | sudo tee -a /etc/hosts
   ```

2. **Navigate to the staff control panel (SCP):**
   ```
   http://support.inlanefreight.local/scp/login.php
   ```
   The SCP login page is the admin/agent interface — separate from the customer-facing portal at `/open.php` or `/tickets.php`.

### Phase 2 — Authentication

3. **Log in with known credentials** — in this lab, the creds come from earlier enumeration or are provided in the module reading: `kevin@inlanefreight.local:Fish1ng_s3ason!`

### Phase 3 — Information Harvesting

4. **Check closed tickets** — once logged in, click on **"Closed"** in the ticket view. Closed tickets are goldmines because staff often consider them "done" and forget sensitive data was exchanged in them.

5. **Read ticket threads** — look for passwords, internal hostnames, API keys, or any sensitive data shared between support agents and customers. In this lab, a thread posted on 9/23/21 at 7:55 PM contains a password sent from a Customer Support Agent to customer Charles Smithson.

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Password sent from support agent to Charles Smithson | Inlane_welcome! | Login to SCP at `/scp/login.php` with `kevin@inlanefreight.local:Fish1ng_s3ason!` → Closed tickets → read the thread |

## Key Takeaways

- **osTicket's value is in the data, not the exploit** — there's no RCE here; the attack is accessing the portal and mining it for sensitive info
- **Closed tickets are higher value than open ones** — staff and customers are less careful in older exchanges, and passwords/configs are often shared in the clear
- **Support portals are social engineering goldmines** — even without staff creds, some osTicket instances allow public ticket submission with email-based registration, which can be abused for social engineering or to register email addresses on other services (the module reading covers this with the `@inlanefreight.local` domain trick)
- **SCP vs customer portal** — `/scp/login.php` is the staff panel (full ticket access); the customer portal at the root only shows that customer's own tickets
- **Credentials found here chain into other attacks** — a password like `Inlane_welcome!` found in a ticket could be reused across other services (password reuse / credential stuffing)

## Gotchas

- The SCP login URL is `/scp/login.php` — don't confuse it with the customer-facing login at the root
- Tickets may have multiple thread entries — scroll through all of them; the sensitive data may not be in the first message
- Email-based registration on osTicket can be abused to receive confirmation emails on `@inlanefreight.local` addresses, potentially allowing registration on other internal services that use email verification
