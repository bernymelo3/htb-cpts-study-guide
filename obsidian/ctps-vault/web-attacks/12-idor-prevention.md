# THEORY — IDOR Prevention

## ID
212

## Module
Web Attacks

## Kind
notes

## Title
Section 12 — IDOR Prevention

## Description
Covers the two-layer defense against IDOR vulnerabilities: implementing object-level access control (RBAC) and replacing predictable object references with strong random identifiers like UUIDs.

## Tags
theory, idor, access-control, rbac, defensive, prevention

## TL;DR — What's Important
- **Fix access control first, references second:** A solid RBAC system is the primary defense; opaque references are a hardening layer on top, not a replacement.
- **Never trust client-side role data:** Map privileges from the back-end session token, not from cookies or request parameters the user controls.
- **Use UUID v4 or salted hashes as object references:** Sequential IDs (`uid=1`, `uid=2`) make enumeration trivial; random 128-bit UUIDs make it practically impossible.
- **Never generate or expose hashes on the front end:** Compute them server-side at object creation time and store the mapping in the database.
- **UUIDs alone don't fix IDOR:** They raise the bar for discovery but a broken access control system is still exploitable (e.g., replaying one user's request with another user's session).

## Concept Overview
IDOR vulnerabilities stem from two combined weaknesses: broken access control and predictable object references. Prevention therefore requires two layers: a robust object-level access control system that validates every request against the requester's privileges, and strong, non-guessable references that prevent enumeration even if a control gap exists.

## Key Concepts

### Comparison Table
| Defense Layer | What It Solves | Example |
|---|---|---|
| Object-Level Access Control (RBAC) | Ensures the requester is authorized for the specific object | Back-end checks `user.uid == requestedUid \|\| user.roles == 'admin'` per request |
| Strong Object References (UUID/hash) | Prevents enumeration and guessing of valid object IDs | `89c9b29b-d19f-4515-b2dd-abb6e693eb20` instead of `uid=1` |

### Definitions
- **RBAC (Role-Based Access Control)** — A system where each user is assigned a role with defined privileges; every request is checked against those privileges before granting access.
- **Object-Level Access Control** — Per-object authorization checks that verify the requesting user has the right to read/write/delete that specific resource.
- **UUID v4** — A 128-bit randomly generated identifier with ~5.3 × 10³⁶ possible values, making brute-force guessing infeasible.

### Categories / Types
- **Insecure patterns:** Storing roles in cookies or user-controlled fields; using sequential integer IDs; computing hashes client-side.
- **Secure patterns:** Mapping roles from back-end session tokens; generating UUIDs server-side at creation; maintaining a DB cross-reference table between UUIDs and internal objects.

## Why It Matters
On real engagements you'll encounter applications that rely solely on obscurity (random-looking URLs) with no back-end authorization checks — those are still exploitable via session replay. Conversely, apps with solid RBAC but sequential IDs leak their entire object space to enumeration. Both layers must be present.

## Defender Perspective
- **Detection:** Monitor for sequential or high-volume requests to object endpoints (e.g., iterating `uid=1` through `uid=10000`); alert on access-denied spikes.
- **Mitigations:** Centralize authorization in middleware/gateway rather than per-endpoint; enforce deny-by-default; audit all API endpoints for missing access checks.
- **MITRE ATT&CK:** Relates to T1078 (Valid Accounts) when attackers leverage session tokens, and generally to Exploitation of Application Vulnerabilities.

## Key Takeaways
- Access control must be validated on every single request at the object level — not just at login or page load.
- Replacing clear-text references with UUIDs is a hardening step that makes IDOR harder to discover and exploit, but it is not a fix on its own.
- Front-end hash generation is a false sense of security because the algorithm and inputs are visible to the attacker.
- Even with UUIDs, techniques like replaying another user's session or swapping tokens still bypass the reference layer, which is why RBAC is the primary control.

## Gotchas
- An application can appear "safe" during testing simply because UUIDs are hard to guess — this doesn't mean access control is actually enforced. Always test by swapping session tokens between two valid users.
- Hashing sequential IDs (e.g., MD5 of `uid=1`) is not the same as UUID v4 — if the attacker knows the algorithm they can pre-compute the entire range.
