# NOTE — .well-known URIs

## ID
213

## Module
Information Gathering – Web Edition

## Kind
theory

## Title
Section 14 — .Well-Known URIs

## Description
RFC 8615 standardises a `/.well-known/` directory at the site root for service metadata: `security.txt`, `change-password`, `openid-configuration`, `assetlinks.json`, `mta-sts.txt`, etc. Hitting these URIs is a fast way to map services and discover OIDC endpoints.

## Tags
well-known, rfc-8615, openid-connect, security-txt, mta-sts, oidc, recon, metadata

## TL;DR — What's Important
- `/.well-known/<thing>` is a standardised metadata location at the site root (RFC 8615).
- IANA maintains the registry of allowed names.
- Best ones for recon: `security.txt` (vuln contact), `openid-configuration` (full OIDC endpoint dump), `mta-sts.txt` (mail security policy), `assetlinks.json` (mobile app/domain links).
- Always probe the IANA registry of `.well-known` URIs — each one is a free metadata leak.

## Concept Overview
The `/.well-known/` standard centralises service metadata at a predictable location. Browsers, apps, and security tools can auto-discover config without guessing paths. For pentesters, that same predictability means consistent, low-effort intel gathering.

## Common URIs in the IANA Registry
| URI suffix | Purpose | Status |
|---|---|---|
| `security.txt` | Vuln-disclosure contact info | Permanent (RFC 9116) |
| `change-password` | Standard URL for password change page | Provisional |
| `openid-configuration` | OIDC provider metadata | Permanent |
| `assetlinks.json` | Verifies digital asset ownership (mobile apps) | Permanent |
| `mta-sts.txt` | SMTP MTA Strict Transport Security policy | Permanent (RFC 8461) |

## The OIDC Goldmine — `openid-configuration`
If a site uses OpenID Connect, hitting `/.well-known/openid-configuration` returns a JSON dump of every endpoint and capability. Example response:
```json
{
  "issuer": "https://example.com",
  "authorization_endpoint": "https://example.com/oauth2/authorize",
  "token_endpoint": "https://example.com/oauth2/token",
  "userinfo_endpoint": "https://example.com/oauth2/userinfo",
  "jwks_uri": "https://example.com/oauth2/jwks",
  "response_types_supported": ["code", "token", "id_token"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "scopes_supported": ["openid", "profile", "email"]
}
```

What it gives you:
| Field | Recon use |
|---|---|
| `authorization_endpoint` | Where users authorize — entry point for OAuth abuse |
| `token_endpoint` | Where tokens are issued — fuzz for misconfigured grant types |
| `userinfo_endpoint` | What user data the server exposes |
| `jwks_uri` | Public signing keys — for JWT validation/forgery research |
| `scopes_supported` | What permissions exist — informs privilege-escalation paths |
| `id_token_signing_alg_values_supported` | Algorithm details — `none` or `HS256` algs are exploit candidates |

## Recon Workflow
1. For each domain, request `/.well-known/security.txt` (vuln-disclosure contact) — useful even outside attacks (legit way to report findings).
2. Probe `/.well-known/openid-configuration` — full OIDC endpoint dump if applicable.
3. Probe `/.well-known/change-password` — auto-redirect to password change page (sometimes reveals auth backend).
4. Probe `/.well-known/assetlinks.json` — confirms linked mobile apps.
5. Probe `/.well-known/mta-sts.txt` — mail security policy hints at email infra.
6. Browse the **IANA Well-Known URI Registry** for additional candidates.

## Key Takeaways
- `/.well-known/` is a structured, free metadata leak — always probe it.
- `openid-configuration` is the highest-value one when present — gives you the entire OIDC attack surface.
- `security.txt` doubles as your professional vuln-disclosure path.
- Many endpoints are provisional/uncommon — IANA registry is the source of truth.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-recon]]
← [[13-robots-txt]] | [[15-creepy-crawlies]] →
<!-- AUTO-LINKS-END -->
