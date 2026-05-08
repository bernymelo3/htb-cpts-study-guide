## ID
113

## Module
Attacking Common Services

## Kind
notes

## Title
Section 13 — Attacking DNS

## Description
Enumerate DNS servers, attempt zone transfers (AXFR) on base domain and discovered subdomains, and extract flags from TXT records – a complete internal DNS assessment workflow.

## Tags
dns, enumeration, zone-transfer, axfr, subdomain, subbrute, txt-record

## Commands
- nmap -p53 -Pn -sV -sC <TARGET_IP>
- dig AXFR @<TARGET_IP> <DOMAIN>
- subfinder -d <DOMAIN> -v
- python3 subbrute.py <DOMAIN> -s ./names.txt -r ./resolvers.txt
- fierce --domain <DOMAIN>
- host <SUBDOMAIN>.<DOMAIN>
- curl -s "https://crt.sh/?q=%25.<DOMAIN>&output=json" | jq -r '.[].name_value' | sort -u

## What This Section Covers
DNS enumeration starts with checking for zone transfers (AXFR) on the base domain. If that fails, subdomain enumeration is performed (using subbrute with the target as resolver). Each discovered subdomain is then tested for zone transfers. TXT records often contain flags or sensitive information.

## Methodology

1. **Initial enumeration** – Nmap to confirm DNS service on port 53 (UDP/TCP).
2. **Zone transfer on base domain** – `dig AXFR @target domain`. If successful, the flag may be in any record.
3. **Subdomain enumeration** – Use subbrute with the target DNS server as the resolver (works in internal networks without internet).
4. **Zone transfer on each subdomain** – For every discovered subdomain, attempt `dig AXFR`.
5. **Extract flag** – Look for TXT records in successful zone transfers.

## Multi‑step Workflow

```bash
# Step 1 – Confirm DNS service
nmap -p53 -Pn -sV -sC 10.129.4.191

# Step 2 – Try AXFR on base domain (fails)
dig AXFR @10.129.4.191 inlanefreight.htb
# Output: Transfer failed.

# Step 3 – Subdomain enumeration using subbrute with target as resolver
git clone https://github.com/TheRook/subbrute.git
cd subbrute
echo "10.129.4.191" > ./resolvers.txt
python3 subbrute.py inlanefreight.htb -s ./names.txt -r ./resolvers.txt
# Found: hr.inlanefreight.htb

# Step 4 – Zone transfer on the discovered subdomain
dig AXFR @10.129.4.191 hr.inlanefreight.htb
# Output includes a TXT record with the flag

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|common-services]]  
← [[11-attacking-rdp]] | [[14-dns-latest-vulns]] →
<!-- AUTO-LINKS-END -->
