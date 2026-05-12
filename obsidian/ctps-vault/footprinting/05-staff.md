# NOTE — Staff

## ID
17

## Module
Footprinting

## Kind
notes

## Title
Section 5 — Staff

## Description
Use LinkedIn (and Xing), public job postings, and employee GitHub profiles to infer the company's tech stack, security posture, and find leaked secrets in personal repos.

## Tags
osint, linkedin, github, job-postings, tech-stack, social-engineering, staff

## TL;DR — What's Important
- Job postings = a public dump of the company's stack: languages, frameworks, DBs, CI/CD, ticketing, version control.
- Employee GitHub repos sometimes contain personal email + hardcoded secrets (JWTs, API keys, DB creds).
- Look for **security-team** members specifically — their skills tell you what defences are in place.

## Concept Overview
Employees market themselves online (LinkedIn, GitHub, blogs) and in doing so leak the tech footprint of their employer. Job postings are the most concentrated form: HR has to enumerate the exact tools required to do the job. Combined with the named hires you find on LinkedIn, you can build a high-confidence picture of the internal stack.

## What to Mine
| Source | What you learn |
|--------|----------------|
| **LinkedIn job posts** | Languages (Java/C#/Python/PHP/Perl/Ruby), DBs (Postgres/MySQL/MSSQL/Oracle), web frameworks (Flask/Django/Spring/ASP.NET), DevOps (Git/SVN/Perforce, Docker/K8s, CI), Atlassian suite, security certs (Sec+) |
| **LinkedIn employee profiles** | Skills, roles ("VP SWE" → leads what), open-source links |
| **GitHub repos linked from LinkedIn** | Personal email, project names, hardcoded JWTs/keys, internal tool clones |
| **Xing** (DACH region) | Same data, different audience |
| **Conference talks / blog posts** | What the security team actively works on |

## How Job Postings Help
A single post enumerates:
- Programming languages preferred
- Database engines used (MySQL/Postgres/MSSQL/Oracle)
- Web frameworks (Flask, Django, ASP.NET, Spring) → look up OWASP cheatsheets for each
- ORM (SQLAlchemy, Hibernate, Entity Framework) — implies SQLi attack surface shape
- Test frameworks → CI/CD pipeline implied
- VCS (Git/SVN/Mercurial/Perforce) → look for exposed `.git` directories later
- Atlassian suite (Confluence, Jira, Bitbucket) — try them at known internal hostnames
- Cloud + container stack
- Security certs required → maps to defensive posture

## Why Mining Security-Role Listings Matters
> "Based on the security area and the employees who work in that area, we will also be able to determine what security measures the company has put in place to secure itself."

A "Senior SOC Analyst, Splunk + CrowdStrike + Tenable" listing tells you exactly what stack is watching you.

## Personal-Repo Footguns (real precedent)
- README/`package.json` lists employee's personal email → confirms identity + opens phishing target.
- Test files contain a hardcoded JWT for the *company's* dev environment — re-usable.
- Older repos still contain credentials that have not been rotated.

## Lab — Questions & Answers
*(No lab in this section.)*

## Key Takeaways
- LinkedIn search filters (location, company, title, school) compose into very tight targeting.
- Job postings are an unfiltered tech-stack dump — read multiple postings, intersect for the canonical stack.
- Always pivot from LinkedIn to the employee's GitHub. Personal repos leak more than people realise.

## Gotchas
- LinkedIn rate-limits / shadow-bans aggressive scraping — use sparingly or use an alt account.
- Old job postings persist on third-party indexes (Indeed, archive.org) even after removal — check there.
- Employees move on; a stack listed in a 3-year-old post may have been replaced. Cross-reference against current postings.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[04-cloud-resources]] | [[06-ftp]] →
<!-- AUTO-LINKS-END -->
