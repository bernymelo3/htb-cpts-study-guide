## ID
703

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 26 — Attacking LDAP

## Description
Covers LDAP fundamentals, how LDAP differs from Active Directory, LDAP injection techniques using wildcard and special characters, and bypassing authentication on an OpenLDAP-backed web app.

## Tags
ldap, injection, authentication-bypass, wildcard, openldap, nmap

## Commands
- nmap -p- -sC -sV --open --min-rate=1000 <TARGET_IP>
- ldapsearch -H ldap://<TARGET_IP>:389 -D "cn=admin,dc=example,dc=com" -w <PASSWORD> -b "ou=people,dc=example,dc=com" "(mail=john.doe@example.com)"

## What This Section Covers
LDAP is a protocol for accessing and managing hierarchical directory services (users, groups, devices). When web applications use LDAP for authentication without sanitizing input, attackers can inject special characters into login fields to manipulate LDAP queries — analogous to SQL injection but targeting directory services. This section covers LDAP fundamentals, the distinction between LDAP and Active Directory, LDAP injection mechanics, and practical exploitation by injecting wildcards to bypass authentication.

## Methodology
1. Enumerate the target: `nmap -p- -sC -sV --open --min-rate=1000 <TARGET_IP>` — look for port 389 (LDAP) or 636 (LDAPS) alongside a web server on port 80/443
2. If OpenLDAP is running alongside a web app, assume the app likely uses LDAP for authentication
3. Test for LDAP injection by entering `*` as both the username and password on the login form
4. The wildcard `*` matches any entry in the LDAP directory, causing the authentication query to return a valid result and granting access

## Key Takeaways
- LDAP commonly runs on port 389 (plaintext) or 636 (LDAPS/SSL) — nmap identifies it as `OpenLDAP 2.2.X - 2.3.X` or similar
- LDAP ≠ Active Directory: LDAP is a **protocol** for querying directories; AD is a **directory service** that uses LDAP as one of its protocols alongside Kerberos and DNS
- LDAP injection special characters to test: `*` (wildcard), `()` (grouping), `|` (OR), `&` (AND), `(cn=*)` or `(objectClass=*)` (always-true conditions)
- A typical vulnerable LDAP auth query looks like: `(&(objectClass=user)(sAMAccountName=$username)(userPassword=$password))` — injecting `*` into either field makes the filter match all entries
- LDAP does **not** encrypt traffic by default — credentials sent over port 389 are plaintext unless LDAPS or StartTLS is used
- **Defense:** sanitize input by stripping LDAP special characters (`*`, `(`, `)`, `|`, `&`, `\`), use parameterized queries, and enforce LDAPS

## Gotchas
- LDAP injection with `*` only works if the application passes raw input directly into the LDAP filter — properly parameterized queries are immune
- Don't confuse LDAP injection with LDAP enumeration (e.g., `ldapsearch` for recon) — injection targets the application layer, enumeration targets the LDAP service directly
- Some apps may filter `*` but not other injection characters like `)(cn=*` — try multiple payloads

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| After bypassing the login, what is the website "Powered by"? | w3.css | Visible on the page after LDAP injection bypass with `*` as username and password |
