# NOTE — Web Attacks § 10: IDOR in Insecure APIs

## ID
534

## Module
Web Attacks

## Kind
notes

## Title
Section 10 — IDOR in Insecure APIs

## Description
Demonstrates IDOR information disclosure in a REST API where GET requests to the profile endpoint lack access control, leaking other users' details including uuid and role — setting up for chained IDOR attacks.

## Tags
idor, api, information-disclosure, rest-api, access-control, json

## Commands
- curl -s "http://<TARGET>:<PORT>/profile/api.php/profile/<UID>" | python3 -m json.tool
- curl -s -X PUT "http://<TARGET>:<PORT>/profile/api.php/profile/<UID>" -H "Content-Type: application/json" -d '{"uid":<UID>,"uuid":"<UUID>","role":"<ROLE>","full_name":"<NAME>","email":"<EMAIL>","about":"<TEXT>"}'
- curl -s -X POST "http://<TARGET>:<PORT>/profile/api.php/profile/<UID>" -H "Content-Type: application/json" -d '{...}'
- curl -s -X DELETE "http://<TARGET>:<PORT>/profile/api.php/profile/<UID>"

## What This Section Covers
REST APIs that use standard HTTP methods (GET/PUT/POST/DELETE) for CRUD operations may apply access control inconsistently across methods. This section tests all four methods against the profile API: PUT has uid/uuid mismatch checks, POST/DELETE require admin role, but GET has no access control at all — allowing any user to read any other user's full profile by changing the uid in the endpoint path.

## Methodology
1. Intercept the Edit Profile flow in Burp — observe the PUT request to `/profile/api.php/profile/1` with JSON body containing uid, uuid, role, full_name, email, about
2. Note the `role=employee` cookie and `"role": "employee"` in the JSON — access control is partially client-side
3. Test PUT with a different uid → `uid mismatch` (backend validates uid matches endpoint)
4. Test PUT with a different user's endpoint + their uid → `uuid mismatch` (backend validates uuid belongs to that user)
5. Test POST to create a user → `Creating new employees is for admins only`
6. Test DELETE → `Deleting employees is for admins only`
7. Send the intercepted GET request to Burp Repeater, change `/profile/1` to `/profile/5`, and send → returns full profile with no access check
8. This leaks the target user's uuid, role, and other details — enabling chained attacks

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — uuid of uid=5 | eb4fe264c10eb7a528b047aa983a4829 | Intercepted GET request in Burp, sent to Repeater, changed endpoint to `/profile/api.php/profile/5` |

## Key Takeaways
- APIs often enforce access control on write operations (PUT/POST/DELETE) but forget read operations (GET) — always test all HTTP methods separately
- Hidden JSON parameters (uid, uuid, role) in PUT requests reveal the API's internal data model — use this to understand what's available
- Client-side role assignment (`Cookie: role=employee`) is a red flag: it suggests the backend may trust client-supplied role values
- Information disclosure IDORs are the first step in chained attacks: leak uuid → use it to bypass uuid mismatch checks on PUT
- The REST convention (GET=read, PUT=update, POST=create, DELETE=remove) tells you exactly which operations to test

## Gotchas
- The profile page sends both a POST and a GET request on load — make sure you intercept the GET to Repeater, not the POST
- The API returns JSON but without pretty-printing; pipe through `python3 -m json.tool` or `jq` for readability
- Changing the uid in the JSON body without also changing the endpoint path gives `uid mismatch` — both must match
