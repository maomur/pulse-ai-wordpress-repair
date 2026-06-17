# CLAUDE.md — Kido WP Repair

Guidance for working in this repo.

## What this is
An AI **skill called "Kido WP Repair"** that cleans hacked/compromised WordPress sites (files **and** database) and produces a professional report dashboard.

- Display/brand name: **Kido WP Repair**.
- Internal skill slug & GitHub repo: `maomur/pulse-ai-wordpress-repair` (kept as-is so `npx skills add …` install commands don't break — do not rename without updating README install steps).

## Files
- `SKILL.md` — the remediation workflow (Step 0–7) + dedicated **Database Forensics** step (Step 3B) for a standalone `.sql` dump or a live DB.
- `dashboard-template.html` — self-contained HTML report template. Fill `{{PLACEHOLDERS}}`, duplicate `<!-- REPETIR -->` rows.
- `incident-report-ejemplo.html` — rendered demo with sample data (keep design/markup in sync with the template).
- `README.md` — install + overview.
- `Infectados/` — drop the infected site (`www/`) and DB dump here. **Treat as read-only evidence; never edit.**
- `Curado/` — the cleaned copy + verified DB + the report(s) are delivered here.

## Engagement workflow (how to actually do a job)
1. User puts the infected site in `Infectados/www` and the DB dump (`*.sql`) alongside.
2. Analyze files + DB (static analysis; assume **no php/wp-cli/internet** locally).
3. `cp -a Infectados/www Curado/www`, then remove confirmed malware from the **copy** only.
4. Apply offline-applicable hardening (e.g. block PHP in `uploads/`); verify the copy is clean (IOC sweep = 0).
5. Generate the dashboard from `dashboard-template.html` into `Curado/`, **bilingual ES + EN** (two files), then `open` it for the user.

## Report/dashboard conventions
- Single self-contained HTML (no CDNs). Two modes: **Modo simple** (plain Spanish) + **Modo técnico** (paths/tables/CVEs) — fill both.
- **SF-Symbols-style inline SVG icons**, never emojis. Apple-like design + security-score ring.
- **PDF via native print** (`window.print()`); print CSS dumps both modes. Provide **two buttons: "PDF · Español" and "PDF · English"**; the other-language button links to that language file with `?print=1` (auto-prints on load).
- Brand header/footer "Kido WP Repair". Be **honest with status** — amber + explicit "pending" when the entry point isn't closed or server-side actions remain (don't fake all-green).

## Malware triage notes (from real engagements)
High-confidence IOCs to scan for: `awscloud.icu` (and similar `.icu` C2), `compress.zlib://` drop-in loaders, `curl`+`eval('?>'.$result)` remote-eval, `goto O…` + `gzinflate(base64_decode())` flattened backdoors (marker `data:application/xml;sex64,`), `<?=`+obfuscated `parse_str` shells, password-protected `eval(base64_decode(str_rot13()))` shells, attacker staging dirs `.chunks/` & `.locks/` (numeric subdirs with their own `.htaccess` forcing index.php access), attacker dropper logs (`debug.log`/`LOG_FILE`, often Indonesian).

**Known-good — do NOT delete (false positives seen):** All-In-One WP Security artifacts (`aios-bootstrap.php`, `.user.ini` `auto_prepend_file`, `wp-content/mu-plugins/aios-firewall-loader.php`, `.htaccess` AIOS block); WP core `wp-admin/includes/continents-cities.php` (matches "wso" in city names); `wp-content/languages/*.l10n.php`; linguise cache `.php` (start with `<?php die();`); twig/monolog source containing `str_rot13`.

## Post-cleanup restore troubleshooting (support the user hits)
- **Only sub-pages 404 ("Not Found"), homepage OK** → `.htaccess` missing (hidden file not uploaded). Re-upload it / re-save Permalinks.
- **Everything 404 incl. homepage** → wrong upload nesting (files one level too deep, e.g. `…/www/www/…`). Move contents up to the docroot.
- **Custom login URL 404s** (e.g. AIOS "rename login page" → `/master`): that URL is virtual and **requires `.htaccess`**. Restore `.htaccess` to fix.
- Emergency admin recovery: rename `wp-content/plugins/all-in-one-wp-security-and-firewall` via FTP to disable AIOS and restore `/wp-login.php`.
- Always remind: hidden files `.htaccess` and `.user.ini` must be uploaded (enable "show hidden files").

## Validating dashboard changes
Render with headless Chrome to a screenshot/PDF before claiming it works, e.g.:
`"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless --screenshot=/tmp/x.png --window-size=1180,2200 "file://…"`
and `--print-to-pdf=…` to verify the PDF. macOS `qlmanage -t` makes a quick PDF thumbnail.
