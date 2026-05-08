## ID
840

## Module
Web Attacks

## Kind
notes

## Title
Section 6 — Intro to IDOR

## Description
Explains the nature of Insecure Direct Object References (IDOR), why they are so common, and the serious impact they can have on web applications due to missing access control.

## Tags
theory, idor, access-control, web-attacks, vulnerability

## TL;DR — What's Important
- **IDOR = exposed reference + no access control:** The direct reference (file ID, user ID, etc.) is not the flaw; the lack of a back‑end check that compares the requestor’s permissions to the object is the vulnerability.
- **Missing access control is the root cause:** A solid Role‑Based Access Control (RBAC) system is hard to build, making IDOR one of the most pervasive web vulnerabilities.
- **Impact spans data exposure to full account takeover:** IDOR can lead to viewing private files, modifying or deleting other users’ data, and even calling admin‑only functions to escalate privileges.
- **Front‑end hiding is not security:** Many applications only *display* the user’s own data while all data is technically accessible through direct requests—attackers bypass this with manual HTTP requests.
- **Even major platforms have IDOR:** Facebook, Instagram, and Twitter have all had IDOR bugs because comprehensive access control is genuinely difficult to implement.

## Concept Overview
Insecure Direct Object References arise when a web application uses predictable identifiers (like numeric IDs in URLs or API parameters) to access data, but does not verify that the currently authenticated user is allowed to access the requested object. This section defines IDOR, explains why it’s so prevalent, and outlines the range of impacts—from unauthorised data disclosure to privilege escalation and data manipulation.

## Key Concepts

### What Is an IDOR?
- A direct reference to an internal object, such as a database record, file, or function, is exposed to the end‑user (e.g., `download.php?file_id=123`).
- If the application does not check whether the user *should* be able to access that specific object, simply changing the ID to another value (`file_id=124`) grants access to that object as well.
- The vulnerability is not the exposure itself but the missing or broken access control on the back‑end.

### Why IDOR Is So Common
- Implementing a robust access control system (e.g., RBAC) that covers every resource and every possible request is complex and time‑consuming.
- Automated tools struggle to detect missing access control because they don’t understand the application’s intended permission logic.
- Many developers rely on front‑end restrictions (hiding links or UI elements) without enforcing checks on the server, assuming users will only follow the provided navigation paths.

### Impact of IDOR
- **Information Disclosure:** Accessing other users’ private files, personal data, or confidential documents simply by guessing or enumerating IDs.
- **Data Manipulation:** Modifying or deleting resources that belong to other users (e.g., changing someone else’s email address, deleting their uploads).
- **Privilege Escalation through Insecure Function Calls:** Even if admin‑only functions are hidden, their API endpoints may still accept requests from lower‑privileged users, allowing actions like changing user roles, resetting passwords, or promoting accounts.
- **Account Takeover:** By combining IDOR with other flaws (e.g., resetting a victim’s password via an insecure admin function), an attacker can fully compromise any user account.

## Why It Matters
IDOR is one of the most frequent and high‑impact findings in web application security assessments. Because it often requires only changing a numeric parameter, exploitation is trivial, yet the consequences can be catastrophic—from mass data exfiltration to complete application takeover. Understanding IDOR from both an attacker’s and a defender’s perspective is essential for identifying, exploiting, and fixing these vulnerabilities.

## Defender Perspective
- **Detection:** Monitor for patterns of parameter manipulation in access logs; a user systematically trying sequential IDs or accessing resources they have never touched before is a red flag.
- **Mitigations:**
  - Always perform server‑side authorisation checks: for every request that retrieves or modifies an object, verify the user’s ownership or role permission against that object.
  - Use indirect references (e.g., a random, session‑based token that maps to the actual ID on the server) to break the predictability.
  - Implement and enforce a consistent access control model (RBAC, claims‑based authorization) across the entire application.
  - Never trust that hiding a UI element or restricting navigation is sufficient; enforce access rules independently of the front‑end.
- **MITRE ATT&CK:** IDOR often falls under T1534 (Internal Spearphishing / Account Access Removal) when used to access other accounts, or under T1078 (Valid Accounts) if combined with privilege escalation.

## Key Takeaways
- IDOR is fundamentally an *authorization* problem, not a data‑leakage problem; the fix is robust access control, not merely obfuscating IDs.
- Even a basic, incremental ID scheme is safe **if** every access is verified; the predictability alone is not the risk—the missing check is.
- Testing for IDOR requires a methodical approach: identify all object references, change the identifier with a different user session, and observe whether the application correctly denies access.
- Many “hidden” admin APIs are reachable simply by guessing the endpoint; combining this with IDOR can yield vertical privilege escalation.

## Gotchas
- Some applications hash or encrypt IDs to make them unpredictable, but if the hashing algorithm is reversible or the ID can be derived, the protection is nullified.
- IDOR testing must check not only views but also edit/delete actions—sometimes an application correctly blocks reading but still allows modification.
- Cookies or session tokens may carry user‑specific IDs that, if swapped, cause IDOR without changing URL parameters; always test such scenarios.
- When testing, ensure you use a second, low‑privilege account to avoid confusing self‑access with cross‑user access.