# Section 7 — Drupal - Discovery & Enumeration

## ID
535

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 7 — Drupal - Discovery & Enumeration

## Description
Covers Drupal fingerprinting via CHANGELOG.txt, README.txt, and manifests, enumeration with droopescan, and understanding Drupal's node-based content structure.

## Tags
drupal, cms, enumeration, droopescan, fingerprinting, nodes

## Commands
- `curl -s http://<TARGET>/ | grep Drupal`
- `curl -s http://<TARGET>/CHANGELOG.txt | grep -m2 ""`
- `curl -s http://<TARGET>/CHANGELOG.txt | grep -m1 "Drupal"`
- `droopescan scan drupal -u http://<TARGET>/`

## What This Section Covers
How to identify and enumerate Drupal installations. Drupal is the third most popular CMS (~1.5% of all sites, ~950K instances). The section covers version fingerprinting through multiple files, automated scanning with droopescan (which has much better Drupal support than Joomla support), and understanding Drupal's node-based URL structure (`/node/<id>`).

## Methodology

1. **Confirm Drupal is in use** — check generator meta tag, "Powered by Drupal" footer, or Drupal favicon:
   ```
   curl -s http://drupal.inlanefreight.local | grep Drupal
   ```

2. **Fingerprint the version** via CHANGELOG.txt (blocked on newer installs):
   ```
   curl -s http://drupal-qa.inlanefreight.local/CHANGELOG.txt | grep -m1 "Drupal"
   ```
   If CHANGELOG.txt returns 404, try other methods.

3. **Alternative version sources**:
   - `README.txt`
   - JavaScript files in `media/system/js/`
   - droopescan automated detection

4. **Run droopescan** for version, plugins, and interesting URLs:
   ```
   droopescan scan drupal -u http://drupal.inlanefreight.local/
   ```
   droopescan has much better Drupal support than Joomla — finds plugins, themes, and narrows version ranges.

5. **Check for node content** — Drupal indexes content as nodes at `/node/<id>`. Browse `/node/1`, `/node/2`, etc. to discover content. This URL pattern also helps identify Drupal when custom themes hide other indicators.

## Drupal User Roles

| Role | Capabilities |
|------|-------------|
| Administrator | Complete control over the site |
| Authenticated User | Log in, add/edit articles based on permissions |
| Anonymous | Read posts only (default) |

## Version Fingerprinting Locations

| File/Path | Notes |
|-----------|-------|
| `/CHANGELOG.txt` | Best source — exact version + date. Blocked on newer Drupal 8+ installs |
| `/README.txt` | May reveal version info |
| Page source `<meta name="Generator">` | Shows "Drupal 8" etc. |
| Footer "Powered by Drupal" | Confirms CMS but not version |
| `/node/<id>` URL pattern | Identifies Drupal when themes hide other clues |

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Drupal version on drupal-qa.inlanefreight.local | 7.30 | `curl -s http://drupal-qa.inlanefreight.local/CHANGELOG.txt \| grep -m1 "Drupal"` → "Drupal 7.30, 2014-07-24" |

## Key Takeaways

- **CHANGELOG.txt is the fastest Drupal version check** but newer installs (8+) block access to it — returns 404
- **droopescan is much more capable for Drupal than Joomla** — finds plugins, themes, and version ranges
- **Node-based URLs (`/node/1`)** are a telltale Drupal indicator even with custom themes
- **~950K Drupal instances** exist — less common than WordPress but still frequently encountered, especially in government (56%) and education (23.8%)
- **Drupal login doesn't enumerate users** — generic error message like Joomla

## Gotchas

- Newer Drupal 8+ installs block CHANGELOG.txt and README.txt by default — don't assume the version can't be found, just use droopescan or check JS files
- droopescan may return a range of possible versions (e.g., 8.9.0–8.9.1) — not always exact
