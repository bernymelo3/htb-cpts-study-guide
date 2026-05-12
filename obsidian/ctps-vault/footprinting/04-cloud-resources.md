# NOTE — Cloud Resources

## ID
16

## Module
Footprinting

## Kind
notes

## Title
Section 4 — Cloud Resources

## Description
Find exposed S3 buckets / Azure blobs / GCP storage via Google Dorks, page source, and GrayHatWarfare. Misconfigured public storage leaks docs, credentials, even SSH private keys.

## Tags
cloud, aws, s3, azure, blob, gcp, google-dorks, grayhatwarfare, ssh-keys

## Commands
- Google: `intext:<company> inurl:amazonaws.com`
- Google: `intext:<company> inurl:blob.core.windows.net`
- Google: `intext:<company> inurl:googleapis.com`
- View page source for `<link rel="dns-prefetch">` / `preconnect` — often references blob/CDN hosts
- Search GrayHatWarfare for company name *and* common abbreviations

## Concept Overview
Cloud providers (AWS/Azure/GCP) secure the platform; customers misconfigure their buckets. The default mistake: a storage container made public for one purpose, then forgotten. Public storage may surface in DNS resolution (e.g. `s3-website-us-west-2.amazonaws.com` resolved during subdomain enum) — always note these for the cloud-recon pass.

## Where to Look
| Source | What it gives you |
|--------|-------------------|
| Subdomain → IP resolution | `*.amazonaws.com`, `*.blob.core.windows.net`, `*.storage.googleapis.com` hostnames |
| **Google Dorks** with `inurl:` + `intext:` | Indexed PDFs/docs/code in public buckets |
| **Web page source** (`dns-prefetch`, `preconnect`, `<img src>`) | Bucket/CDN names devs use to offload static content |
| **domain.glass** | Cloudflare/security-vendor verdict + cert + SSL info — extra Gateway-layer intel |
| **GrayHatWarfare** | Indexed AWS/Azure/GCP buckets, filterable by file type |

## Workflow
1. From the subdomain enum results, flag every cloud-provider hostname.
2. Run Google Dorks for the company name across each provider's domain.
3. Scrape the main site's HTML for hardcoded bucket references.
4. Search GrayHatWarfare for the **full name AND abbreviations** — devs name buckets short.
5. Filter by interesting filetypes: `.docx`, `.xlsx`, `.pdf`, `.pem`, `id_rsa`, `.env`, `.config`.

## Worst-case Findings (real precedent in the source)
- `id_rsa` + `id_rsa.pub` left in a public bucket → password-less SSH onto company hosts.
- Customer contracts, employee data, internal pricing decks.
- Source code with hardcoded API keys / DB creds.

## Lab — Questions & Answers
*(No lab in this section.)*

## Key Takeaways
- Treat every `*.amazonaws.com` / `*.blob.core.windows.net` / `*.storage.googleapis.com` discovered during DNS enum as a separate target to fingerprint.
- Always try **abbreviated** versions of the company name on GrayHatWarfare — devs use short bucket names.
- Page source > textual content. Static asset URLs leak bucket names that aren't on the marketing copy.

## Gotchas
- **Don't actively probe / write to** buckets you don't own without contractual permission — even read can be in a grey area depending on jurisdiction. Use third-party indexes (GrayHatWarfare, Google) to stay passive.
- A bucket can be region-specific — `us-west-2` ≠ `us-east-1`. Same name, different region, different ACL.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[03-domain-information]] | [[05-staff]] →
<!-- AUTO-LINKS-END -->
