## ID
850

## Module
Web Attacks

## Kind
notes

## Title
Section 7 — Identifying IDORs

## Description
Practical techniques to discover Insecure Direct Object References by analysing URL parameters, AJAX calls, hashed/encoded references, and comparing data across user roles.

## Tags
idor, discovery, parameter-fuzzing, access-control, web-attacks

## Commands
- `curl -s 'http://<TARGET>/download.php?file_id=2' -H 'Cookie: session=...'`
- `for i in $(seq 1 100); do curl -s "http://<TARGET>/api/user/$i" -H 'Cookie: ...'; done`
- `echo -n 'file_124.pdf' | base64`
- `echo -n 'file_1.pdf' | md5sum`
- `gobuster dir -u http://<TARGET> -w /usr/share/wordlists/dirbuster/directory-list-2.3-small.txt`

## What This Section Covers
This section teaches how to spot IDOR entry points by examining HTTP requests for direct object references (URL parameters, API endpoints, AJAX calls) and understanding how encodings (base64) or hashing (MD5) might be used to obfuscate them. It also covers the technique of registering multiple users and comparing their request patterns to find access control gaps.

## Methodology
1. **Inspect the request:** When you access a resource (file, profile, message), look at the URL, POST body, and headers for parameters that seem to identify the object (e.g., `id=`, `file=`, `uid=`, `filename=`).
2. **Simple increment / fuzzing:**  
   - Change the numeric value (`id=1` → `id=2`) and observe if you get another user’s data.  
   - Use a loop or a tool like Burp Intruder to automate sequential or random ID tests.  
   - For file names, try `file_1.pdf` → `file_2.pdf` or bulk fuzz with a wordlist of common filenames.
3. **Examine front‑end JavaScript:**  
   - Search for AJAX calls that reference endpoints or parameters that aren’t visible in the UI (e.g., `change_user_password`).  
   - These may contain object references (`uid`, `is_admin`) that can be exploited even if hidden from the interface.
4. **Handle encoded/hashed references:**  
   - If a parameter looks like base64 (e.g., `ZmlsZV8xMjMucGRm`), decode it to see the original value, modify it, and re‑encode.  
   - If the reference is a hash (MD5, SHA1), check the JavaScript for how the hash is computed (e.g., `CryptoJS.MD5('file_1.pdf')`) and replicate it for other objects.  
   - Use hash‑identifier tools if the algorithm is not obvious, then attempt known‑plaintext attacks (hash ‘file_1.pdf’ and compare).
5. **Compare across user roles:**  
   - Register two (or more) accounts with different privilege levels.  
   - Capture the requests made by each to access their own data; note the structure and parameters.  
   - Using the session of a low‑privilege user, replay the high‑privilege user’s API call with the same object reference. If it returns data, IDOR is present.  
   - Even if parameters differ, analyse how they are generated – if you can compute another user’s parameter from known data, the system is vulnerable.

## Key Takeaways
- IDOR discovery often starts with simply changing a number, but real‑world apps hide references behind encoding, hashing, or custom schemes; always look deeper.
- JavaScript source code is a goldmine: unused functions and commented‑out AJAX calls often expose admin‑only API endpoints.
- Base64 is not encryption – it’s trivial to decode; if an app relies on it for “security”, assume it’s broken.
- Two‑user comparison is the most reliable way to confirm an IDOR, especially when roles are involved; it eliminates guesswork about which IDs belong to which user.
- Even if an endpoint doesn’t return sensitive data directly, the ability to see that an object exists (or to trigger an action) may be a vulnerability (e.g., user enumeration, password reset poisoning).

## Gotchas
- Some applications use complex, non‑sequential UUIDs (v4) – they are not inherently secure if access control is missing, but guessing is impossible. Focus on finding scenarios where you can obtain other users’ UUIDs (e.g., through public profiles, leaks, or IDOR in a linked resource).
- AJAX endpoints may require specific headers (e.g., `X-Requested-With: XMLHttpRequest`) or CSRF tokens; add them when replaying requests.
- When testing, always use a separate browser profile or incognito session for each user to avoid session contamination.
- Rate‑limiting or account locking might occur when fuzzing IDs aggressively – start slowly and use random delays.
- Check not only GET requests but also POST, PUT, DELETE when modifying resources; IDOR can exist in any HTTP method.