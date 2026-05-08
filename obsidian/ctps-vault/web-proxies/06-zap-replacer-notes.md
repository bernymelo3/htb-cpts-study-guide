
## Lab — Questions & Answers (optional)
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Exercise 1: ZAP Replacer rules for response modification | type="number" → type="text"; maxlength="3" → maxlength="100" | ZAP Replacer → Response Body String |
| Exercise 2: Rule to inject ;ls; into Ping request | Match: `ip` in Request Body String, Replace: `ip;ls;` | ZAP Replacer → Request Body String |

## Key Takeaways
- Match‑and‑replace rules are persistent, unlike manual interception, and apply to every matching request/response automatically.
- Regex matching (`^User-Agent.*$`) is powerful when the exact value is unknown or variable; literal matching is simpler and faster when the exact string is known.
- Response‑body modifications enable bypassing client‑side restrictions (input types, maxlength, disabled fields) permanently for the duration of the test.
- ZAP’s Replacer offers additional control through *Initiators*, letting you restrict rules to specific contexts (e.g., only requests originating from the browser).
- Always check that automatic modifications don't break the application — some changes may cause unexpected behavior if the server relies on certain values.

## Gotchas
- A rule that matches too broadly (e.g., plain `User-Agent` as a literal in the body) can corrupt requests or responses. Use regex anchors (`^…$`) where precision matters.
- Burp’s match‑and‑replace happens silently; if you forget you’ve enabled a rule, you might waste time debugging strange application behaviour.
- Response modification rules will not persist across proxy restarts unless you save your Burp project (Pro) or ZAP session.
- If the server sends `maxlength` as `maxlength="3"` with different quotes or whitespace, an exact literal match will fail — consider regex for flexibility.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-proxies]]  
← [[06-automatic-modification]] | [[07-repeating-requests]] →
<!-- AUTO-LINKS-END -->
