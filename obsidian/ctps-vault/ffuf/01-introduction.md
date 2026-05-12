# THEORY — Introduction

## ID
1

## Module
Ffuf

## Kind
theory

## Title
Section 1 — Introduction

## Description
Module overview: ffuf is the fuzzer of choice for directory, file, vhost, parameter, and value discovery on web apps.

## Tags
ffuf, fuzzing, intro, web, theory

## TL;DR — What's Important
- **Fuzzing = sending lots of candidate inputs and watching the response** (status code, size, words, lines).
- Web servers don't expose a directory of links — fuzzing closes that gap by brute-forcing against a wordlist.
- Status `200` (or `301`/`302`) = it exists. `404` = filtered out by default.
- Five things you'll fuzz across this module: **directories, files/extensions, vhosts, GET/POST parameters, parameter values**.

## Concept Overview
Servers rarely advertise everything they host. Fuzzing scripts try thousands of candidate paths/params from a wordlist and look at the HTTP response to decide whether the candidate exists. ffuf is fast (thousands of req/s), supports recursion, header-injection (for vhost fuzzing), and arbitrary keyword placement via the `FUZZ` token.

## Topics Covered in This Module
- Fuzzing for directories
- Fuzzing for files and extensions
- Identifying hidden vhosts (no public DNS)
- Fuzzing for PHP parameters (GET + POST)
- Fuzzing for parameter values

## Key Takeaways
- ffuf decides "exists vs not" on HTTP code first, then size/words/lines for filtering false positives.
- A 200 with size 0 is suspicious — page exists but is blank/redirected (e.g. `index.php` redirecting elsewhere).
- The `FUZZ` keyword is just a placeholder — you can place it anywhere (URL path, header value, POST body, query string).

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ffuf]]
[[02-web-fuzzing]] →
<!-- AUTO-LINKS-END -->
