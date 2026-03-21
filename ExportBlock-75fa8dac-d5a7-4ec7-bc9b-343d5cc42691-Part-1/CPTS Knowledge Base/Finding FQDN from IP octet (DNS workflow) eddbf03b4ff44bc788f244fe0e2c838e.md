# Finding FQDN from IP octet (DNS workflow)

Category: Enumeration
Command: dig axfr <domain> @<DNS_IP>
dig axfr <subzone> @<DNS_IP>
dnsenum --dnsserver <DNS_IP> --enum -p 0 -s 0 -o <out>.txt -f <wordlist> <subzone>
Description: Logic: identify DNS + base domain → try AXFR on main zone → extract subzones and try AXFR on them → if AXFR denied, brute-force subdomains with a focused list → match A records whose IP ends with the required octet; the corresponding hostname is the FQDN to submit.
Tags: enumeration

## DNS recon with dig (one-page cheatsheet)

1. **Basic lookups**
    - Default A lookup:
        - `dig example.com`
    - Ask a specific DNS server:
        - `dig @10.10.10.10 example.com`
    - Short answers (great for scripts):
        - `dig example.com +short`
2. **Common record types**
    - IPv4 / IPv6:
        - `dig A example.com`
        - `dig AAAA example.com`
    - Mail servers:
        - `dig MX example.com`
    - Nameservers:
        - `dig NS example.com`
    - TXT (SPF, verification tokens, etc.):
        - `dig TXT example.com`
    - CNAME:
        - `dig CNAME www.example.com`
3. **SOA – Start of Authority**
    - What it is: one per zone, metadata about that DNS zone.
    - Commands:
        - `dig SOA example.com`
        - `dig SOA example.com @<DNS_IP>`
    - Gives you:
        - Primary NS for the zone
        - Admin email (`hostmaster.example.com` → `hostmaster@example.com`)
        - Serial, refresh, retry, expire, minimum TTL
    - Use it to confirm the zone exists and see who is authoritative.
4. **AXFR – Zone transfers**
    - What it is: full zone transfer (primary → secondary), abused for recon.
    - Try to dump the main zone:
        - `dig AXFR example.com @<DNS_IP>`
    - Try sub-zones discovered from the main zone:
        - `dig AXFR internal.example.com @<DNS_IP>`
        - `dig AXFR dev.example.com @<DNS_IP>`
    - If it works, you get:
        - All (or most) records in the zone
        - Subdomains, internal hosts, sometimes internal IP ranges
    - Always try AXFR on:
        1. The main domain
        2. Any subdomains that look like separate zones (internal, dev, corp, etc.)
5. **“Show me whatever you’ll give” (ANY) and version**
    - ANY (often limited on modern servers, but worth trying):
        - `dig ANY example.com @<DNS_IP>`
    - BIND version (CHAOS class):
        - `dig CH TXT version.bind @<DNS_IP>`
6. **Reverse lookups (PTR)**
    - IP → hostname:
        - `dig -x 10.129.9.231`
        - `dig -x 10.129.9.231 @<DNS_IP>`
    - Use when you have IPs (especially internal) and want hostnames.
7. **Output control (clarity/scripting)**
    - Only the answer section:
        - `dig example.com +noall +answer`
    - Short answers:
        - `dig example.com +short`
    - Force TCP (useful for some servers and for AXFR):
        - `dig AXFR example.com @<DNS_IP> +tcp`
8. **Typical pentest workflow with dig**
    1. Basic info:
        - `dig example.com +short`
        - `dig NS example.com +short`
        - `dig MX example.com +short`
        - `dig TXT example.com +short`
    2. Check zone metadata:
        - `dig SOA example.com`
    3. Try info-leaky queries:
        - `dig ANY example.com @<DNS_IP>`
        - `dig CH TXT version.bind @<DNS_IP>`
    4. Attempt zone transfers:
        - `dig AXFR example.com @<DNS_IP>`
        - Then for any discovered sub-zones:
            - `dig AXFR internal.example.com @<DNS_IP>`
            - `dig AXFR dev.example.com @<DNS_IP>`
    5. If AXFR fails, brute-force subdomains with tools (dnsenum, gobuster dns, etc.).
    6. Use `dig -x` on interesting IPs to find hostnames.