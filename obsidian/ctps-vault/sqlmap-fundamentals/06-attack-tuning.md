# NOTE — Attack Tuning

## ID
401

## Module
SQLMap Essentials

## Kind
notes

## Title
Section 6 — Attack Tuning

## Description
Covers fine-tuning SQLMap's detection with `--level`, `--risk`, custom `--prefix`/`--suffix`, UNION tuning (`--union-cols`, `--union-char`, `--union-from`), and advanced response comparison switches (`--code`, `--titles`, `--string`, `--text-only`, `--technique`).

## Tags
sqlmap, sqli, level-risk, prefix-suffix, union-tuning, attack-tuning

## Commands
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?<PARAM>=*' --level 5 --risk 3 -T <TABLE> --batch --dump
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?<PARAM>=<VALUE>' --prefix='<PREFIX>' -T <TABLE> --batch --dump
- sqlmap -u 'http://<TARGET>:<PORT>/<PAGE>?<PARAM>=<VALUE>' --technique=U --union-cols=<NUM> --batch --dump
- sqlmap -u '<URL>' --prefix='<PREFIX>' --suffix='<SUFFIX>' --batch --dump
- sqlmap -u '<URL>' -v 3 --level=5 --risk=3
- sqlmap -u '<URL>' --technique=BEU --batch --dump
- sqlmap -u '<URL>' --code=200 --batch --dump
- sqlmap -u '<URL>' --string='success' --batch --dump

## What This Section Covers
SQLMap's default settings (level 1, risk 1) use ~72 payloads per parameter — enough for most cases. When the injection point has non-standard SQL syntax (unusual boundaries), requires OR-based payloads (login pages), or needs UNION with a known column count, you need to tune the attack. Cranking `--level 5 --risk 3` expands to ~7,865 payloads but is much slower — use it surgically, not as a default.

## Methodology

1. **Try default settings first** — always start with a basic SQLMap run. If it finds the injection, you're done. Only tune when the default fails.

2. **Raise `--level` and `--risk` for stubborn injection points** — `--level` (1–5) increases the boundary (prefix/suffix) combinations tested; `--risk` (1–3) adds riskier payload types like OR-based blind. Use `--level 5 --risk 3` when the default run finds nothing but you're confident the param is injectable:
   `sqlmap -u 'http://<TARGET>:<PORT>/case5.php?id=*' --level 5 --risk 3 -T flag5 --batch --dump`

3. **Use `--prefix` / `--suffix` for non-standard boundaries** — if you know the SQL context wraps your input in unusual syntax (e.g. the query uses `` `) `` to close), supply it directly instead of relying on SQLMap to guess:
   `sqlmap -u 'http://<TARGET>:<PORT>/case6.php?col=id' --prefix='`)' -T flag6 --batch --dump`

4. **Tune UNION injection with `--technique=U` and `--union-cols`** — if you already know the column count (e.g. from manual testing or page output), tell SQLMap directly to skip guessing:
   `sqlmap -u 'http://<TARGET>:<PORT>/case7.php?id=1' --technique=U --union-cols=5 -T flag7 --batch --dump`

5. **Use `-v 3` to debug payloads** — shows every `[PAYLOAD]` being sent, useful for understanding what boundaries/vectors are being tried and why detection is failing.

6. **Narrow technique type with `--technique`** — letters map to: `B` = Boolean blind, `E` = Error-based, `U` = UNION, `S` = Stacked, `T` = Time-based blind. Combine as needed (e.g. `--technique=BEU` skips time-based and stacked).

7. **Advanced response comparison** — when TRUE/FALSE responses are hard to distinguish:
   - `--code=200` — fixate TRUE detection on a specific HTTP status code
   - `--string='success'` — detect TRUE by a specific string present in response
   - `--titles` — compare responses by `<title>` tag content
   - `--text-only` — strip all HTML tags, compare visible text only

## When to Use What — Decision Logic

### Case #5 → `--level 5 --risk 3` (brute-force all payloads)

**What you see:** A GET parameter `id` in the URL. You run default SQLMap and it finds nothing — "all tested parameters do not appear to be injectable."

**Why it fails:** The injection point sits inside unusual SQL syntax on the server. SQLMap's default 72 payloads don't include the specific prefix/suffix combo that matches how your input is embedded in the query. You don't know what the server-side SQL looks like, so you can't supply a custom prefix.

**When to use this approach:** You're confident the param is injectable (the lab tells you, or you got a weird error in manual testing) but you don't know the exact SQL context. You tell SQLMap "try everything" — all 7,865 payload/boundary combos. One of them will match.

**Trade-off:** Slow. Only do this after the default run fails. Never start here.

### Case #6 → `--prefix` (you know the SQL boundary)

**What you see:** A `col` parameter that controls column sorting (like `ORDER BY col`). Default SQLMap fails. Even `--level 5 --risk 3` might not find it because the boundary is non-standard.

**Why it fails:** The server's SQL wraps the column name in a backtick and parenthesis — something like `` ORDER BY `col`) ``. SQLMap needs to close that `` `) `` before injecting its vector, but that specific combo isn't in its boundary list. The debug output (`-v 3`) shows it skipping tests because the level isn't high enough, or trying wrong boundaries.

**When to use this approach:** You can infer the SQL context from error messages, source code, or the way the parameter is used (column name in ORDER BY = likely backtick-quoted). You manually tell SQLMap exactly how to close the existing SQL with `--prefix='`)'` — backtick closes the column quoting, paren closes the grouping. This is like picking the lock instead of trying every key.

**Trade-off:** Requires you to understand the server-side SQL context. Faster and more precise than level/risk brute-force.

### Case #7 → `--technique=U --union-cols=5` (surgical UNION)

**What you see:** The page displays query results in a visible table (rows and columns on screen). This means UNION-based injection is possible — you can append your own `SELECT` and its output renders on the page.

**Why default might be slow:** SQLMap tries every technique (boolean blind, error-based, time-based, etc.) before getting to UNION, and then guesses the column count by trying `UNION SELECT NULL`, then `NULL,NULL`, etc. You already know it's UNION-viable and the query has 5 columns.

**When to use this approach:** You can see tabular output on the page (confirms UNION works) and you know or can determine the column count (count the visible columns, or test manually with `ORDER BY 5` → works, `ORDER BY 6` → fails = 5 columns). You skip all the guessing with `--technique=U` (only try UNION) and `--union-cols=5` (go straight to 5 columns).

**Trade-off:** Fastest approach but requires you to do some manual recon first. If the column count is wrong, it silently fails.

### Summary: Which flag to reach for

| Situation | Flag(s) | Speed |
|---|---|---|
| Default run fails, you don't know why | `--level 5 --risk 3` | Slow (7,865 payloads) |
| You know the SQL boundary / quoting style | `--prefix` / `--suffix` | Fast (surgical) |
| Page shows tabular output + you know column count | `--technique=U --union-cols=N` | Fastest (direct hit) |
| Login page needs OR-based payloads | `--risk 3` (minimum) | Medium |
| You want to debug what SQLMap is trying | `-v 3` | Same speed, more output |

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Contents of table flag5? (Case #5) | `HTB{700_much_r15k_bu7_w0r7h_17}` | Default fails → unknown SQL context → brute-force: `sqlmap -u 'http://<TARGET>:<PORT>/case5.php?id=*' --level 5 --risk 3 -T flag5 --batch --dump` |
| Q2 — Contents of table flag6? (Case #6) | `HTB{v1nc3_mcm4h0n_15_4570n15h3d}` | Default fails → `col` param is backtick+paren quoted → supply prefix: `sqlmap -u 'http://<TARGET>:<PORT>/case6.php?col=id' --prefix='`)' -T flag6 --batch --dump` |
| Q3 — Contents of table flag7? (Case #7) | `HTB{un173_7h3_un173d}` | Page shows tabular output → UNION viable, 5 columns visible → surgical: `sqlmap -u 'http://<TARGET>:<PORT>/case7.php?id=1' --technique=U --union-cols=5 -T flag7 --batch --dump` |

## Key Takeaways
- Default level 1 / risk 1 uses ~72 payloads per param. Level 5 / risk 3 jumps to ~7,865 — only use it when the default fails.
- `--risk 3` adds OR-based payloads which are dangerous on production — they can modify data via UPDATE/DELETE statements. Never use high risk on live targets without understanding this.
- If you know the SQL boundary context (from source code, error messages, or manual testing), feeding `--prefix` / `--suffix` directly is faster and more reliable than cranking level/risk.
- `-T <TABLE>` dumps only the specified table — much faster than `--dump` which grabs everything.
- `--technique=U --union-cols=<N>` is the surgical approach when you've already confirmed UNION works and know the column count.
- `--union-char='a'` can help when the default NULL fill values cause issues with the target query's expected data types.

## Gotchas
- **Using `--level 5 --risk 3` as a default habit** — massively slower and the OR payloads can break things. Always start with defaults, escalate only when needed.
- **Forgetting `-T` when you only need one table** — without it, SQLMap dumps every table in the database, wasting time on labs and potentially causing issues on real targets.
- **Wrong `--prefix` syntax** — the prefix must match exactly how the SQL query closes before your injection point. Get it wrong and nothing works. Check error messages or source code for clues.
- **UNION column count mismatch** — if `--union-cols` is wrong, the UNION injection silently fails. Confirm the count manually first (e.g. `ORDER BY N` or `UNION SELECT 1,2,3...` in browser).


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|sqlmap-fundamentals]]  
← [[05-handling-errors]] | [[07-database-enumeration]] →
<!-- AUTO-LINKS-END -->
