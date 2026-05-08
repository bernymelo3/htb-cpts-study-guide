# NOTE — Web Attacks § 11: Chaining IDOR Vulnerabilities

## ID
535

## Module
Web Attacks

## Kind
notes

## Title
Section 11 — Chaining IDOR Vulnerabilities

## Description
Chains an IDOR information disclosure (GET leak of admin uid/uuid) with an IDOR insecure function call (PUT to modify admin email) to take over an admin account and retrieve the flag.

## Tags
idor, chained-attack, api, privilege-escalation, uuid-leak, burp-suite

## Commands
- for uid in {1..10}; do curl -s "http://<TARGET>:<PORT>/profile/api.php/profile/$uid"; echo; done
- curl -s "http://<TARGET>:<PORT>/profile/api.php/profile/<UID>" | jq .
- curl -s -X PUT "http://<TARGET>:<PORT>/profile/api.php/profile/<UID>" -H "Content-Type: application/json" -d '{"uid":"<UID>","uuid":"<UUID>","role":"<ROLE>","full_name":"<NAME>","email":"flag@idor.htb","about":"<TEXT>"}'

## What This Section Covers
Individual IDOR vulnerabilities may seem low-impact — reading another user's profile or hitting a uuid mismatch on writes. But chaining an information disclosure IDOR (leaking uuid via GET) with an insecure function call IDOR (modifying details via PUT) enables full account takeover. This section demonstrates the complete chain: enumerate users, find the admin, leak their uuid, then use it to modify their email.

## Methodology
1. Enumerate users with a bash loop: GET `/profile/api.php/profile/{1..10}` and grep for `admin`
2. Identify the admin account — note uid (`10`), uuid (`bfd92386a1b48076792e68b596846499`), and role (`staff_admin`)
3. In the browser, go to Edit Profile and click "Update profile" while Burp intercepts
4. In Burp, modify the intercepted PUT request:
   - Change the endpoint from `/profile/1` to `/profile/10`
   - Replace the JSON body with the admin's details, changing only the email to `flag@idor.htb`
5. Forward the request — no access control error because the uuid now matches
6. Refresh the Edit Profile page — the flag appears at the bottom

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Change admin email to `flag@idor.htb` | HTB{1_4m_4n_1d0r_m4573r} | Enumerated admin uid/uuid via GET IDOR, then PUT to `/profile/10` with leaked uuid and modified email, flag rendered on profile page |

## Key Takeaways
- The attack chain pattern: **enumerate → leak → exploit** — GET to find the admin's uuid, PUT to use it
- The uuid mismatch check only validates that the uuid belongs to the target uid — it doesn't verify the *requester* is that user
- The `role=employee` cookie is the only authorization; the backend never validates who is actually making the request
- Once you have `staff_admin` role, you can also create/delete users via POST/DELETE — full CRUD takeover
- In real engagements, the same chain enables password reset hijacking (change email → trigger reset) or stored XSS (inject payload in about field)

## Gotchas
- You must use the admin's actual uuid in the PUT body — your own uuid will trigger `uuid mismatch`
- The endpoint path AND the uid in the JSON body must both be changed to the target user's uid
- The flag only appears after refreshing the Edit Profile page — it doesn't show in the Burp response to the PUT request
- The admin role in this lab is `staff_admin`, not `admin` or `web_admin` — role names vary per lab instance
