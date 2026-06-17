---
name: pulse-ai-wordpress-repair
description: "Pulse AI WordPress Repair - Clean and repair a hacked or compromised WordPress site, including forensic analysis of the database (live or a standalone .sql dump). Use when the user mentions: wordpress hacked, wordpress infected, wordpress malware, wordpress virus, wordpress repair, wordpress cleanup, wordpress blacklisted, wordpress redirect hack, wordpress pharma hack, wordpress phishing, wordpress defaced, wordpress backdoor, wordpress suspicious files, wordpress spam, wordpress mu-plugins malware, wordpress supply chain attack, wordpress fake admin, wordpress reinfection, wordpress database hacked, wordpress database malware, infected sql dump, wp_options malware, malware wordpress, sitio wordpress hackeado, wordpress infectado, limpiar wordpress, reparar wordpress, wordpress comprometido, wordpress se reinfecta, admin falso wordpress, base de datos wordpress infectada, analizar base de datos wordpress, dump sql infectado. Guides through full remediation: assessment, backup, file + database malware scan, cleanup, hardening, and monitoring."
metadata:
  tags: wordpress, security, malware, repair, cleanup, hack, backup, hardening, remediation, mu-plugins, supply-chain, backdoor, wpscan, incident-response, database-forensics, sql-dump, wp_options
  platforms: Claude, ChatGPT, Gemini
  version: 2.2.0
---

# Pulse AI WordPress Repair

Complete workflow for cleaning and repairing a hacked or compromised WordPress site. Follow every step in order — skipping steps leads to re-infection.

## How to use this skill

The intended flow is: **the user points you at an infected WordPress directory; you analyze it, apply the fixes, and produce a clear dashboard report.**

1. The user gives you a path (e.g. `/home/user/public_html` or a local copy of the site) and, ideally, the domain.
2. Run the workflow below (Steps 0–7) against that directory, collecting **real data** as you go — every file you find, every change you make. Keep a running log; you will need it for the report.
3. **Deliverable:** at the end, generate a self-contained HTML dashboard ("**Kido WP Repair**") from [`dashboard-template.html`](dashboard-template.html) — see Step 7. The dashboard has two views: **👤 Modo simple** (plain language for a non-technical owner) and **🔧 Modo técnico** (tables, paths, commands, CVEs). Fill BOTH, and (when the owner needs it) deliver **two language versions, ES + EN**, with cross-linked PDF buttons.

**Folder convention (preserve evidence):** if the user provides an `Infectados/` (infected) folder, treat it as **read-only forensic evidence — never edit it**. Copy the site to a `Curado/` (cleaned) folder (`cp -a Infectados/www Curado/www`) and do all removal/hardening on the copy. Deliver the cleaned site, the verified DB, and the dashboard inside `Curado/`. Assume **no php / wp-cli / internet** when working a local copy — rely on static analysis (`grep`/`find`/`diff`).

> **Golden rule of 2025/2026:** If the site gets re-infected after a "clean", you missed a **persistence mechanism** (backdoor, malicious cron job, mu-plugin, drop-in, or `wp-config.php` injection). The cleanup is not done until the site survives **30 days** without reinfection. Watch closely for the first **48–72 hours**.

---

## Threat Landscape (2025–2026) — read this first

These facts shape where you look and what you prioritize:

- **Plugins are the #1 entry point.** ~91% of new WordPress vulnerabilities are in **plugins** (not core, not themes). Core itself is rarely the hole.
- **Exploitation is fast.** Median time from public disclosure to active exploitation is now **~5 hours**. An outdated plugin is a live wire, not a future risk.
- **The biggest classes** are XSS (~35% of all reports), **Broken Access Control / privilege escalation**, and **arbitrary file upload (RCE)**. Many of the worst 2025 CVEs let an **unauthenticated** attacker register as / promote themselves to **administrator** (e.g. role manipulation on registration forms, exposed REST API bearer tokens, auth bypass).
- **Supply-chain attacks are now mainstream.** Legitimate plugins get **sold, hijacked, or have their update servers compromised**, then push malware to every site in a routine update. "I only use plugins from wordpress.org" is no longer a guarantee.
- **Persistence has moved beyond `*.php` shells in uploads/.** Modern hiding spots you MUST check: `wp-content/mu-plugins/` (auto-loaded, no activation needed), **drop-ins** (`object-cache.php`, `advanced-cache.php`, `db.php`, `maintenance.php`), **malicious WP-Cron events**, **`wp-config.php` injections that re-download malware**, fake admin users, and `auto_prepend_file` directives in `.user.ini` / `php.ini` / `.htaccess`.
- **Evasion is sophisticated.** C2 domains hidden in blockchain smart contracts, multi-stage obfuscation, and patches that fix the vuln **but do not remove already-injected code**. Updating a plugin does NOT clean an infection — you must still hunt the payload.

---

## Quick Reference: Common Hack Types

| Hack Type | Symptoms | Typical Entry / Persistence |
|-----------|----------|------------------------------|
| **Redirect Hack** | Visitors redirected to spam/porn/pharma sites | Malicious JS in theme, DB, or mu-plugin; conditional on referrer/mobile |
| **Pharma / SEO Spam** | Search engines show drug/pill/casino keywords | SEO spam injected in DB, cloaked content, sitemap poisoning |
| **Backdoor** | Attacker regains access after cleanup | Hidden PHP in mu-plugins, drop-ins, core dirs; `eval()`, malicious cron, wp-config injection |
| **Defacement** | Homepage replaced with attacker's message | File write via vulnerable plugin / upload flaw |
| **Phishing** | Fake login/bank pages served from your site | Files uploaded to uploads/ or random subfolders |
| **Malicious Admin** | Unknown admin users in wp-admin | Privilege-escalation CVE, brute force, stolen creds, registration role abuse |
| **Cryptominer** | Server CPU spikes, slow site | Injected JS (browser miner) or PHP script |
| **Mailer Spam** | Server blacklisted for sending spam | PHP mailer script in uploads/ or mu-plugins |
| **Supply-chain** | Many sites infected at once after a plugin update | Hijacked/sold plugin or compromised update server |

---

## Workflow

### Step 0: Triage & Authorization (before touching anything)

- [ ] Confirm you have **authorization** to work on this site (you, the owner, or a written engagement).
- [ ] Get access details: SSH/SFTP, hosting panel (cPanel/Plesk), and whether **WP-CLI** is available (`wp --info`). Most steps below assume WP-CLI.
- [ ] Note the **WordPress path** (the docroot) and **domain**. Throughout this skill, replace `/path/to/wordpress/` and `yourdomain.com` with the real values.
- [ ] Establish a **clean workstation** — never log in to the infected wp-admin from a machine you don't trust; stolen session cookies are a thing.

> If WP-CLI is not installed, you can run a portable copy:
> ```bash
> curl -O https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
> php wp-cli.phar --info
> # then use:  php wp-cli.phar <command> --allow-root --path=/path/to/wordpress
> ```

---

### Step 1: Initial Assessment

**Identify the hack type and scope:**

```bash
# Check Google Safe Browsing status
# Open in browser: https://transparencyreport.google.com/safe-browsing/search?url=yourdomain.com

# Check Sucuri SiteCheck (free, remote scanner)
# Open in browser: https://sitecheck.sucuri.net/results/yourdomain.com

# Check VirusTotal URL reputation
# Open in browser: https://www.virustotal.com/gui/home/url  (paste the URL)

# View live source for visible injections (also test with a Googlebot / mobile user-agent —
# many redirects are cloaked and only fire for search engines or mobile visitors)
curl -sL "https://yourdomain.com" | grep -E "(eval|base64_decode|unescape|fromCharCode|atob|document\.write)" | head -20
curl -sL -A "Googlebot" "https://yourdomain.com" | grep -iE "(viagra|cialis|casino|<script src)" | head -20
curl -sL -A "Mozilla/5.0 (iPhone)" "https://yourdomain.com" | grep -iE "(window\.location|<iframe|<script src)" | head -20

# Recently modified files (great first signal — narrow the window to the incident date)
find /path/to/wordpress/ -type f -name "*.php" -mtime -10 | head -50
find /path/to/wordpress/ -type f -name "*.js"  -mtime -10 | head -30
```

**Scan against the public vulnerability databases** to find the likely entry point (do this early — it tells you which plugin let them in):

```bash
# WPScan CLI (needs a free API token from wpscan.com). Enumerates version, vulnerable
# plugins/themes, and users. Use --enumerate vp,vt,u  (vuln plugins, vuln themes, users)
wpscan --url https://yourdomain.com --enumerate vp,vt,u --api-token YOUR_TOKEN

# No WPScan? Cross-check installed versions manually against:
#   https://wpscan.com/wordpresses/  •  https://patchstack.com/database/  •  https://www.wordfence.com/threat-intel/
wp plugin list --fields=name,status,version,update --allow-root
```

**Assess infection scope:**
- [ ] Homepage visually defaced?
- [ ] Redirects happening? (test mobile, incognito, and with a Googlebot user-agent — see above)
- [ ] Google / Bing showing warnings or spam in search results (`site:yourdomain.com`)?
- [ ] Hosting provider suspended or warned the account?
- [ ] Unknown admin users in wp-admin?
- [ ] Spam emails being sent from the server / domain on a blocklist?
- [ ] Sudden CPU/bandwidth spikes (cryptominer or mailer)?

---

### Step 2: Backup & Isolation

**CRITICAL: Always make a forensic backup before any changes — even of an infected site. You need the evidence and a rollback point.**

```bash
# === BACKUP VIA WP-CLI ===
wp db export backup-$(date +%Y%m%d-%H%M%S).sql --allow-root
tar -czf backup-files-$(date +%Y%m%d-%H%M%S).tar.gz /path/to/wordpress/

# === BACKUP VIA cPanel / Plesk ===
# cPanel > Backup Wizard > Full Backup   OR   phpMyAdmin > Export database

# Store backups OFF the server (download them). Label them "INFECTED — evidence".

# === PUT SITE IN MAINTENANCE MODE (contain the attack, stop serving malware) ===
wp maintenance-mode activate --allow-root
# OR manually:
echo "<?php \$upgrading = time(); ?>" > /path/to/wordpress/.maintenance
```

**Rotate ALL credentials immediately** — assume everything is compromised:

```bash
# Reset every WordPress user's password (forces attacker sessions out once keys rotate)
wp user list --field=ID --allow-root | xargs -I {} wp user update {} --user_pass="$(openssl rand -base64 18)" --allow-root

# Rotate WordPress secret keys/salts — this invalidates all stolen login cookies/sessions
wp config shuffle-salts --allow-root   # WP-CLI 2.x; otherwise paste new keys from the URL below
# Manual: replace the 8 define() lines from https://api.wordpress.org/secret-key/1.1/salt/

# Log everyone out for good measure
wp user session destroy --all --all-users --allow-root 2>/dev/null || true
```

**Also change immediately (outside WordPress):**
- [ ] Hosting panel (cPanel/Plesk) password
- [ ] **All FTP/SFTP/SSH accounts** (attackers create extra FTP users — audit and delete unknown ones)
- [ ] Database password (and update `wp-config.php`)
- [ ] Email accounts associated with the domain
- [ ] Any API keys / tokens stored in the site (payment, SMTP, cloud storage)

---

### Step 3: Scan for Malware

Run **all** of these. The classic "PHP files in uploads/" check is necessary but no longer sufficient — modern backdoors hide in mu-plugins, drop-ins, cron, and the database.

```bash
# === CORE & PLUGIN INTEGRITY (fast, high-signal) ===
wp core verify-checksums --allow-root
wp plugin verify-checksums --all --allow-root
# Any "File doesn't verify against checksum" or "should not exist" line = tampered. Investigate each.

# === PHP WHERE IT SHOULD NOT EXIST ===
find /path/to/wordpress/wp-content/uploads/ -type f \( -name "*.php" -o -name "*.phtml" -o -name "*.php5" -o -name "*.phar" \)
find /path/to/wordpress/ -type f -name "*.php" -newer /path/to/wordpress/wp-config.php | grep -v "/.git/"

# === MODERN PERSISTENCE SPOTS (the ones people miss) ===
# mu-plugins: auto-loaded, no activation, a top backdoor location in 2025
ls -la /path/to/wordpress/wp-content/mu-plugins/ 2>/dev/null
# A legit site usually has 0 files (or a known few). ANY unexpected .php here is suspect.

# Drop-ins: special filenames in wp-content/ that WP auto-loads
ls -la /path/to/wordpress/wp-content/{object-cache.php,advanced-cache.php,db.php,maintenance.php,sunrise.php} 2>/dev/null
# These exist legitimately only with specific caching/multisite plugins. Verify each one's origin.

# === COMMON MALWARE CODE PATTERNS (signatures) ===
PATTERNS='eval\(|base64_decode|gzinflate|gzuncompress|str_rot13|create_function|assert\(|preg_replace.*/e|call_user_func|\$_(POST|GET|REQUEST|COOKIE)\[.*\]\(|FilesMan|c99shell|r57shell|WSO|wso_|edoced_46esab|move_uploaded_file|file_put_contents.*\$_|system\(|shell_exec|passthru|popen|proc_open'
grep -rlnE "$PATTERNS" /path/to/wordpress/ --include="*.php" | grep -v "/.git/"
# Also catch \x-escaped and char-concatenated obfuscation:
grep -rlnE '(\\x[0-9a-f]{2}){6,}|chr\([0-9]+\)\.chr\(' /path/to/wordpress/ --include="*.php"

# Hidden / oddly-named shells
find /path/to/wordpress/ -name ".*.php" -o -name "...php" 2>/dev/null
find /path/to/wordpress/ -name "wp-*.php" ! -path "*/wp-includes/*" ! -path "*/wp-admin/*" -maxdepth 2

# === REAL-WORLD IOCs (seen in live engagements) ===
# Remote-eval C2 loaders, compress.zlib drop-in loaders, goto-flattened packers, attacker staging dirs
grep -rlaE "compress\.zlib:|eval\('\?>'\s*\.|goto O[0-9]{5,}|application/xml;sex64|\.icu['\"/]" /path/to/wordpress/ --include="*.php" 2>/dev/null
# Hidden attacker staging directories at the docroot (numeric subdir + its own .htaccess forcing index.php)
ls -la /path/to/wordpress/.chunks /path/to/wordpress/.locks 2>/dev/null
# Attacker dropper logs left behind (often Bahasa Indonesia: "berhasil", "menyimpan file", "dihapus")
grep -rlaiE "berhasil|menyimpan file|memproses metode" /path/to/wordpress/ 2>/dev/null

# auto_prepend_file backdoors (run code on EVERY request, survive plugin cleanup)
grep -rniE "auto_prepend_file|auto_append_file" /path/to/wordpress/ --include=".user.ini" --include="*.ini" --include=".htaccess" 2>/dev/null
cat /path/to/wordpress/.user.ini 2>/dev/null

# === wp-config.php — read it fully; look for injected includes / fetches ===
grep -nE "eval|base64|include|require|file_get_contents|curl_|gzinflate|fopen|\$_(GET|POST)" /path/to/wordpress/wp-config.php
# Some 2025 backdoors re-inject wp-config.php from a remote server on a schedule — see Step 4.

# === MALICIOUS WP-CRON EVENTS (persistence that re-downloads malware) ===
wp cron event list --allow-root
# Look for events with random/garbled hook names or hooks pointing at unknown functions.

# === DATABASE ===
# Injected redirects / scripts in options
wp db query "SELECT option_name FROM wp_options WHERE option_value LIKE '%eval(%' OR option_value LIKE '%base64_decode%' OR option_value LIKE '%<script%' OR option_value LIKE '%fromCharCode%';" --allow-root
# SEO / pharma spam in content
wp db query "SELECT ID, post_title FROM wp_posts WHERE post_content RLIKE 'viagra|cialis|casino|payday|<script|eval\\\(';" --allow-root
# Backdoors stashed in postmeta / usermeta
wp db query "SELECT post_id FROM wp_postmeta WHERE meta_value LIKE '%eval(%' OR meta_value LIKE '%base64_decode%';" --allow-root
# siteurl/home tampering (a classic for redirect/takeover)
wp option get siteurl --allow-root ; wp option get home --allow-root

# === FAKE / HIDDEN ADMIN USERS (check BOTH WP-CLI and raw DB — some are hidden from wp-admin) ===
wp user list --role=administrator --fields=ID,user_login,user_email,user_registered --allow-root
wp db query "SELECT u.ID,u.user_login,u.user_email,u.user_registered FROM wp_users u
  JOIN wp_usermeta m ON u.ID=m.user_id
  WHERE m.meta_key='wp_capabilities' AND m.meta_value LIKE '%administrator%';" --allow-root
# Audit Application Passwords (a quiet persistent-access method)
wp db query "SELECT user_id, meta_value FROM wp_usermeta WHERE meta_key='_application_passwords';" --allow-root
```

> **Tip:** Comparing against a known-clean copy beats signatures. Download the exact same versions of WP core, the active theme, and each plugin into a temp dir and `diff -r` against the live install — any extra or modified file jumps out, even with zero malware signatures.

> ⚠️ **Known-good — do NOT delete (false positives seen in the field).** Verify before removing:
> - **All-In-One WP Security (AIOS):** `aios-bootstrap.php`, the `.user.ini` `auto_prepend_file` line pointing to it, `wp-content/mu-plugins/aios-firewall-loader.php`, and the `# BEGIN All In One WP Security` block in `.htaccess` are **legitimate**. (Confirm `aios-bootstrap.php` only includes the AIOS firewall — not remote/eval code.)
> - WP core **`wp-admin/includes/continents-cities.php`** matches naive patterns (e.g. `wso` inside city names) but is pure data.
> - **`wp-content/languages/*.l10n.php`** (compiled translations), and **third-party source** (twig/monolog) that legitimately contains `str_rot13`.
> - Plugin caches that start with `<?php die();` (e.g. **linguise** `…/cache/*.php`) are guarded data, not shells.
> - A renamed/numbered admin user (e.g. `admin4896`) is often the **AIOS-renamed original admin**, not a rogue — verify its email/registration date before deleting.

---

### Step 3B: Database Forensics (standalone `.sql` dump OR live DB)

The database is a first-class infection target, not an afterthought. In 2025 `wp_options` is the **#1 hiding spot**: attackers stash base64 blobs under hex/random option names, abuse `autoload='yes'` so the payload runs on **every page load**, and hide code in **transients** and the **`cron`** option. Other injections live in `wp_posts` (JS/iframe, SEO spam, hidden posts), `wp_usermeta` (rogue/hidden admins, app passwords), serialized `*meta`, and **snippet plugins** (e.g. WPCode) that store executable PHP in the DB. Run a full DB pass even if the files look clean — a DB-only reinfector will rebuild the file backdoors.

You will often be handed a **standalone database export** (`backup.sql`, `dump.sql.gz`). You can analyze it **two ways**:

#### Path A — Static analysis of the dump file (no DB server needed, safest first pass)

```bash
DUMP=/path/to/database.sql               # if gzipped: gunzip -k database.sql.gz  (or use zcat/zgrep)

# 0) Overview: charset, table prefix (NOT always wp_!), and table list
grep -m1 -iE "CREATE DATABASE|SET NAMES|DEFAULT CHARSET" "$DUMP"
grep -oE "CREATE TABLE \`[^\`]+\`" "$DUMP"        # note the real prefix; attackers also ADD tables here
grep -oE "INSERT INTO \`[^\`]+\`" "$DUMP" | sort -u

# 1) Code-execution / injection signatures anywhere in the dump
grep -anoiE "eval\(|base64_decode|gzinflate|gzuncompress|str_rot13|create_function|assert\(|<script|<iframe|document\.write|fromCharCode|String\.fromCharCode|\\\\x[0-9a-f]{2}|preg_replace.*/e" "$DUMP" | head -50
# Obfuscation that AVOIDS literal eval/base64 (concatenation, reversed strings): scan visually
grep -anoiE "edoced_46esab|\bchr\(|\.\s*chr\(|gzuncompress\(base64|call_user_func" "$DUMP" | head -30

# 2) wp_options — the prime target
#    a) Hex/garbage option_name + base64 value (classic backdoor list of spam URLs)
grep -anoE "INSERT INTO \`[a-z0-9_]*options\`[^;]*" "$DUMP" | grep -iE "[a-f0-9]{16,}|base64|<script|eval\(" | head
#    b) siteurl / home pointing somewhere unexpected (redirect/takeover)
grep -anoE "'(siteurl|home)',\s*'[^']*'" "$DUMP"
#    c) Malicious scheduled tasks live inside the serialized 'cron' option
grep -anoE "'cron'[^;]{0,4000}" "$DUMP" | grep -oiE "https?://[^\"'\\\\]+|[a-z0-9_]+(check|update|class|wp_)[a-z0-9_]*" | head
#    d) Snippet plugins storing PHP (WPCode/Code Snippets) — inspect their option/table rows
grep -aniE "wpcode|code_snippets|custom_css_js|insert_php" "$DUMP" | head

# 3) wp_users / wp_usermeta — rogue & HIDDEN admins
grep -anoE "INSERT INTO \`[a-z0-9_]*users\`[^;]*" "$DUMP"            # list every user row + email
grep -anoiE "wp_capabilities[^;]{0,120}administrator|wp_user_level[^;]{0,20}10" "$DUMP" | head
grep -aniE "_application_passwords|session_tokens" "$DUMP" | head    # persistent-access tokens

# 4) wp_posts / wp_comments — SEO spam, injected markup, hidden content
grep -anoiE "viagra|cialis|casino|payday|essay writing|<script|<iframe|\.ru/|\.tk/|\.top/" "$DUMP" | head -40

# 5) Unknown tables the attacker created (anything outside the standard WP set + your known plugins)
```

#### Path B — Load into a sandbox DB and query (precise, serialized-aware)

```bash
# Spin up a THROWAWAY local DB (never your production server) and import the dump
mysql -u root -p -e "CREATE DATABASE wp_forensics;"
mysql -u root -p wp_forensics < "$DUMP"          # gunzip first if needed
# Then query (adjust the wp_ prefix to the real one found above):
mysql -u root -p wp_forensics <<'SQL'
SELECT option_id,option_name,autoload,LEFT(option_value,80)
  FROM wp_options
  WHERE option_value REGEXP 'eval\\(|base64_decode|<script|<iframe|fromCharCode'
     OR option_name REGEXP '^[a-f0-9]{16,}$';
SELECT u.ID,u.user_login,u.user_email,u.user_registered
  FROM wp_users u JOIN wp_usermeta m ON u.ID=m.user_id
  WHERE m.meta_key='wp_capabilities' AND m.meta_value LIKE '%administrator%';
SELECT ID,post_status,post_type,post_title
  FROM wp_posts
  WHERE post_content REGEXP '<script|<iframe|base64_decode|viagra|casino';
SELECT meta_id,post_id,LEFT(meta_value,80) FROM wp_postmeta
  WHERE meta_value REGEXP 'eval\\(|base64_decode|gzinflate';
SQL
```

For the **live** site, the same queries run via `wp db query "..."` / `wp option list --search='*'` / `wp user list` (see the WP-CLI quick checks in Step 3). On the live site, **prefer disabling first, deleting second**: `wp option update <opt> ... ` or set `autoload` to `no` to neutralize a suspicious option before removing it.

**What to flag as malicious:**
- `wp_options` rows with hex/random names, base64/serialized blobs, `<script>`/`<iframe>`, or `autoload='yes'` on anything you don't recognize.
- `siteurl`/`home` not matching the real domain; a `cron` option referencing unknown hooks or external URLs.
- Admin users you can't account for — **cross-check `wp_users` against `wp_usermeta` capabilities**; a user present in `usermeta` as administrator but odd/absent in `users` is a red flag.
- Any executable PHP/JS stored via snippet plugins; spam in posts/comments; attacker-created tables.

> ⚠️ **Serialized-data warning (critical):** WordPress stores serialized PHP arrays (e.g. `a:2:{s:5:"...";...}`) where each string is prefixed by its **byte length**. A naive SQL `REPLACE()` or `sed` on those values **corrupts the length counts and breaks the site**. For any replacement that touches serialized data (most `wp_options`, widget, and `*meta` edits) use a **serialization-aware** tool:
> - Live DB: `wp search-replace 'old' 'new' --all-tables --report-changes-only` (add `--dry-run` first).
> - Dump file: a serialized-safe replacer such as **WP Search Replace CLI** or **interconnect/it Search-Replace-DB**, then re-import.
> Deleting a whole offending row/option is safe; editing *within* a serialized value is not — use the safe tools.

---

### Step 4: Remove Malware

```bash
# === REINSTALL CLEAN CORE (does NOT touch wp-content or wp-config.php) ===
wp core download --force --allow-root
wp core verify-checksums --allow-root   # must pass now

# === REMOVE MALICIOUS FILES ===
find /path/to/wordpress/wp-content/uploads/ -type f \( -name "*.php" -o -name "*.phtml" \) -delete
# Remove confirmed backdoors found in Step 3 (mu-plugins / drop-ins / shells) — by exact path:
# rm /path/to/wordpress/wp-content/mu-plugins/<malicious>.php
# rm /path/to/wordpress/wp-content/object-cache.php   # only if confirmed malicious
# rm /path/to/suspicious-shell.php

# === KILL PERSISTENCE: wp-config.php injection ===
# 1) Manually remove any injected eval/include/remote-fetch block from wp-config.php.
# 2) Search the whole tree for the loader that re-creates it, and remove that too:
grep -rlnE "wp-config" /path/to/wordpress/ --include="*.php" | xargs grep -lE "file_put_contents|fwrite|curl_|file_get_contents" 2>/dev/null

# === KILL PERSISTENCE: malicious cron ===
wp cron event delete <malicious_hook> --allow-root   # for each bad event from Step 3

# === CLEAN THE DATABASE (always back up first — see Step 3B for full forensics) ===
# SAFE within serialized values: use the serialization-aware tool, dry-run first.
wp search-replace '<INJECTED_SNIPPET>' '' --all-tables --dry-run --report-changes-only --allow-root
wp search-replace '<INJECTED_SNIPPET>' '' --all-tables --report-changes-only --allow-root
# Deleting a whole malicious row/option is safe (no serialized-length issue):
wp option delete <malicious_hex_option_name> --allow-root
wp db query "DELETE FROM wp_postmeta WHERE meta_value LIKE '%eval(base64_decode%';" --allow-root
# Neutralize-before-delete: stop a suspicious option running on every request
wp option update <suspect_option> --autoload=no --allow-root 2>/dev/null || true
# Reset siteurl/home if tampered
# wp option update siteurl "https://yourdomain.com" --allow-root
# wp option update home    "https://yourdomain.com" --allow-root
# NOTE: a plain SQL REPLACE() on serialized data corrupts byte-length prefixes — prefer wp search-replace.

# === REMOVE FAKE ADMINS & STALE ACCESS ===
# wp user delete <BAD_ID> --reassign=1 --allow-root
# Revoke ALL application passwords (regenerate only the ones you actually use)
wp db query "DELETE FROM wp_usermeta WHERE meta_key='_application_passwords';" --allow-root

# === CLEAN .htaccess / drop-ins ===
cat /path/to/wordpress/.htaccess   # remove injected RewriteRules / redirects / php_value auto_prepend_file
wp rewrite flush --allow-root      # regenerate clean WordPress rules
```

> **Multisite / many sites on one account:** clean ALL sites in the same hosting account together. Cross-site reinfection through a shared parent directory is extremely common — one missed site re-infects the rest.

---

### Step 5: Plugin & Theme Audit (close the door)

Cleaning files is pointless if the hole is still open. This step is where you actually stop reinfection.

```bash
# === INVENTORY & DIAGNOSE ===
wp plugin list --fields=name,status,version,update,auto_update --allow-root
wp theme list  --fields=name,status,version,update --allow-root
wp plugin deactivate --all --allow-root        # safe-mode diagnosis

# === REINSTALL EVERYTHING LEGITIMATE FROM OFFICIAL SOURCES (overwrites tampered files) ===
wp plugin install <slug> --force --allow-root   # for each plugin you keep
wp theme  install <slug> --force --allow-root    # for the active theme
# Premium/paid plugins: re-download a fresh copy from the VENDOR, not a "nulled" site.

# === UPDATE TO LATEST (the patched versions) ===
wp core update --allow-root
wp plugin update --all --allow-root
wp theme  update --all --allow-root

# === DELETE WHAT YOU DON'T NEED ===
wp plugin delete <unused-or-abandoned> --allow-root
wp theme  delete <unused-theme> --allow-root      # keep one default theme as a fallback
```

**Manual judgment for each plugin/theme — keep, replace, or remove:**
- [ ] Confirmed entry point (matched a CVE in Step 1)? Update to the patched version, or **remove it** if no patch exists.
- [ ] **Nulled / pirated?** Delete immediately and replace with a licensed copy — nulled software is the single biggest backdoor source. Never "clean and keep" it.
- [ ] Abandoned (no update in >2 years) or removed from the wp.org repo? Replace with a maintained alternative.
- [ ] **Supply-chain risk:** was it recently sold/transferred, or did the infection coincide with a routine update? Check the changelog and recent CVEs; pin/rollback to a known-good version and watch the vendor.
- [ ] **No patch available but you must keep it?** Apply **virtual patching** at the WAF (Wordfence/Patchstack/Cloudflare rules) until a fix ships.

---

### Step 6: Harden Security

```bash
# === FILE PERMISSIONS ===
find /path/to/wordpress/ -type d -exec chmod 755 {} \;     # directories
find /path/to/wordpress/ -type f -exec chmod 644 {} \;     # files
chmod 600 /path/to/wordpress/wp-config.php                 # secrets file
# Ensure files are owned by the web user, NOT writable by group/other where avoidable.
```

```php
// === wp-config.php HARDENING ===
define('DISALLOW_FILE_EDIT', true);   // no theme/plugin editor in wp-admin
define('DISALLOW_FILE_MODS', true);   // optional/strict: blocks installs+updates from admin
define('FORCE_SSL_ADMIN', true);
define('WP_AUTO_UPDATE_CORE', true);  // auto-apply core security updates
define('WP_POST_REVISIONS', 5);
define('WP_DEBUG', false);
define('WP_DEBUG_LOG', false);
define('WP_DEBUG_DISPLAY', false);
// Confirm the salts/keys were rotated in Step 2.
```

```apacheconf
# === .htaccess HARDENING (Apache) — add ABOVE "# BEGIN WordPress" ===
# Block PHP execution in uploads/ (stops uploaded shells from running) — single most valuable rule
<Directory "/path/to/wordpress/wp-content/uploads/">
  <FilesMatch "\.(php|phtml|php[0-9]|phar)$">
    Require all denied
  </FilesMatch>
</Directory>

# Protect sensitive files
<FilesMatch "^(wp-config\.php|\.htaccess|\.user\.ini|xmlrpc\.php)$">
  Require all denied
</FilesMatch>

Options -Indexes   # disable directory browsing
```

```bash
# === DISABLE XML-RPC (brute-force & amplification vector) unless you truly need it ===
wp eval 'add_filter("xmlrpc_enabled","__return_false");' --allow-root   # or via security plugin

# === REST API / USER ENUMERATION HARDENING ===
# Block unauthenticated user enumeration via /wp-json/wp/v2/users and ?author=N
# (use a security plugin's toggle, or a small mu-plugin filter). Audit Application Passwords.

# === LOCK DOWN REGISTRATION (the 2025 privilege-escalation theme) ===
wp option get users_can_register --allow-root          # should be 0 unless required
wp option get default_role --allow-root                # MUST be 'subscriber', never higher

# === INSTALL A SECURITY PLUGIN + WAF ===
wp plugin install wordfence --activate --allow-root          # firewall + malware scan + 2FA + login limits
# alternatives: sucuri-scanner (monitoring+WAF), better-wp-security (iThemes/Solid)
```

**Hardening checklist:**
- [ ] **2FA enabled for every admin** (and editor) account — your strongest single control against credential attacks.
- [ ] **WAF in front of the site** — Cloudflare (free tier works) or Wordfence; enables virtual patching + rate limiting.
- [ ] **Limit login attempts** + rename/hide `wp-login.php` (WPS Hide Login).
- [ ] **Principle of least privilege** — give each user the lowest role they need; minimize admins.
- [ ] Rename the default `admin` username to something unique.
- [ ] **Activity / audit logging** (WP Activity Log or Wordfence) so the next incident is traceable.
- [ ] **Auto-updates on** for core and trusted plugins (closes the 5-hour exploitation window).
- [ ] reCAPTCHA on login and comment forms.

---

### Step 7: Verify, Monitor & Report

```bash
# === DEACTIVATE MAINTENANCE MODE (only after the site is verified clean) ===
wp maintenance-mode deactivate --allow-root   # or: rm /path/to/wordpress/.maintenance

# === FINAL RESCAN (everything from Step 3 must now come back clean) ===
wp core verify-checksums --allow-root
wp plugin verify-checksums --all --allow-root
grep -rlnE "eval\(|base64_decode|gzinflate|FilesMan|c99shell" /path/to/wordpress/ --include="*.php" | grep -v "/.git/"
find /path/to/wordpress/wp-content/uploads/ -name "*.php"
ls -la /path/to/wordpress/wp-content/mu-plugins/ 2>/dev/null
wp cron event list --allow-root

# === EXTERNAL CONFIRMATION ===
curl -sL "https://yourdomain.com" | grep -c "<html"
curl -sL -A "Googlebot" "https://yourdomain.com" | grep -iE "(viagra|cialis|casino|<script src)"   # expect no output
# Re-run Sucuri SiteCheck and WPScan; expect clean.

# === REQUEST BLACKLIST REMOVAL (ONLY after the site is fully clean) ===
# Google: Search Console > Security Issues > Request Review  (https://search.google.com/search-console)
# Timeline: typically 1–3 days. Requesting while still infected resets the clock and damages trust.

# === SET UP ONGOING MONITORING & BACKUPS ===
wp plugin install updraftplus --activate --allow-root   # scheduled off-site backups (Drive/Dropbox/S3)
# Enable Wordfence/Sucuri email alerts for file changes, new admins, and logins.
# Uptime + integrity monitoring: UptimeRobot (free). Check Search Console > Security weekly.
```

**Reinfection watch (do not skip):**
- [ ] Re-scan at **+24h, +72h, +7d, +30d**. If anything returns, a persistence mechanism survived — go back to Step 3 and look harder at mu-plugins, drop-ins, cron, wp-config, and other sites on the account.
- [ ] Watch the activity log and new-user notifications for the first 72 hours.

---

**IMPORTANT — Final Deliverable: the Dashboard**

At the end of the remediation you MUST produce a **clear, visual dashboard** as the primary deliverable, plus the structured report below as its data source.

How to generate the dashboard:

1. Copy [`dashboard-template.html`](dashboard-template.html) to `incident-report-<domain>-<YYYYMMDD>.html` in the working directory (or wherever the user wants it).
2. Replace **every** `{{PLACEHOLDER}}` with **real data** from this session. Never leave a `{{...}}` behind; if a datum doesn't apply, write "No aplica" / "No detectado".
3. Duplicate the `<!-- REPETIR -->` template rows once per real item (each malicious file, each DB change, each persistence mechanism, each timeline event, each recommendation).
4. The dashboard MUST stay **self-contained** (no CDNs / external assets) so it opens with a double-click, offline. It is printable to PDF via the browser.
5. Fill **both** layers of meaning:
   - **`.simple` blocks** → plain Spanish a non-technical owner understands ("En palabras simples: …"). No jargon. Explain what happened, what you did, and what they must do now.
   - **`.tech` blocks** → exact file paths in `<code>`, tables, CVE references, commands, checksum results.
6. Set status colors honestly: badge classes `ok` (green), `warn` (amber), `bad` (red), `info` (blue). The hero uses an SF-style SVG icon + tint per the template comments (`i-shieldcheck`+`t-green` = clean, `i-shield`+`t-orange` = clean with caveats, `i-x`+`t-red` = action pending), and a 0–100 security-score ring. Do not show green / a high score if the entry point wasn't closed or reinfection risk remains.
7. Tell the user the file path of the generated dashboard, that it has the simple/technical toggle, and a **"Descargar PDF" button** (top-right) that exports the whole report — both modes, full design — via the browser's "Save as PDF". No edits needed; the button is already wired in the template.

The structured report below defines every field the dashboard needs. Fill it with the actual data collected and actions taken — detailed enough that another technician could understand exactly what happened without asking questions.

The report must include the following sections, fully completed with real data:

---

## 🛡️ WordPress Security Incident Report
**Site:** [domain name]
**Date:** [date of remediation]
**Technician:** Pulse AI WordPress Repair
**Report generated:** [timestamp]

---

### 1. Executive Summary

Write 2–4 sentences summarizing: what happened, when it was discovered, what type of attack it was, what the impact was, and the final status of the site.

---

### 2. Infection Analysis

- **Attack type:** [redirect hack / pharma hack / backdoor / defacement / phishing / malicious admin / cryptominer / spam mailer / supply-chain / other]
- **Confirmed entry point:** [plugin name + version + CVE / nulled plugin / brute force / stolen credentials / registration privilege-escalation / supply-chain (hijacked update) / hosting vulnerability / unknown]
- **Estimated infection date:** [date or "unknown — oldest modified malicious file: filename (date)"]
- **Persistence mechanisms found:** [mu-plugins / drop-ins / malicious cron / wp-config.php injection / auto_prepend_file / fake admin / application passwords / none]
- **Infection scope:**
  - Files affected: [exact list of files found with malicious code]
  - Database tables affected: [e.g., wp_options, wp_posts, wp_postmeta, wp_usermeta]
  - Malware type found: [e.g., eval(base64_decode), PHP shell, JS redirect injector, SEO spam, cryptominer]
- **Blacklist status at start:**
  - Google Safe Browsing: [clean / flagged]
  - Sucuri SiteCheck: [clean / flagged — list detections]
  - VirusTotal: [clean / X engines flagged]

---

### 3. Actions Taken

#### 3.1 Backup
- Backup created: [YES / NO]
- Backup method: [WP-CLI / cPanel / manual]
- Backup location: [path or storage] (stored off-server: [YES/NO])
- Database backup file: [filename]
- Files backup: [filename]

#### 3.2 Malicious Files Removed
| File Path | Issue Found | Action |
|-----------|-------------|--------|
| [/path/to/file.php] | [eval(base64_decode...) / PHP shell / injected JS / mu-plugin backdoor] | Deleted / Cleaned |

#### 3.3 Database Cleanup
| Table | Field | Issue | Action |
|-------|-------|-------|--------|
| wp_options | option_value | Malicious redirect JS | Removed injection |
| wp_posts | post_content | Pharma spam in X posts | Cleaned content |

#### 3.4 Persistence Removed
| Mechanism | Detail | Action |
|-----------|--------|--------|
| Malicious cron event | [hook name] | Deleted |
| wp-config.php injection | [description] | Removed + loader deleted |
| mu-plugin / drop-in | [filename] | Deleted |

#### 3.5 WordPress Core
- Core reinstalled: [YES / NO] — Version: [e.g., 6.7.x]
- Checksum verification result: [PASSED / FAILED — list any failed files]

#### 3.6 Users
- Unknown/malicious admin users removed: [usernames + IDs]
- All user passwords reset: [YES / NO]
- Application passwords revoked: [YES / NO]
- Suspicious accounts: [list or "none found"]

#### 3.7 Plugins & Themes
- Plugins reinstalled from official source: [list]
- Plugins deleted (unused / nulled / abandoned): [list]
- Themes deleted: [list]
- Nulled/pirated software found: [YES — list / NO]
- Supply-chain issue identified: [YES — plugin + detail / NO]
- All plugins/themes updated: [YES / NO — list exceptions + virtual-patch status]

#### 3.8 Credentials Rotated
- [ ] WordPress user passwords  - [ ] WP secret keys/salts  - [ ] Database password
- [ ] FTP/SFTP/SSH accounts (unknown ones deleted)  - [ ] cPanel/hosting  - [ ] Email accounts  - [ ] Stored API keys

---

### 4. Security Hardening Applied

| Measure | Applied | Notes |
|---------|---------|-------|
| DISALLOW_FILE_EDIT / FILE_MODS | ✅ / ❌ | |
| New WordPress secret keys (salts) | ✅ / ❌ | |
| File permissions corrected (755/644/600) | ✅ / ❌ | |
| PHP execution blocked in uploads/ | ✅ / ❌ | |
| .htaccess / sensitive files protected | ✅ / ❌ | |
| XML-RPC disabled | ✅ / ❌ | |
| User enumeration / REST API hardened | ✅ / ❌ | |
| Registration locked (default_role=subscriber) | ✅ / ❌ | |
| Login attempts limited + wp-login hidden | ✅ / ❌ | Plugin: [name] |
| Security plugin installed | ✅ / ❌ | [Wordfence / Sucuri / Solid] |
| WAF + virtual patching | ✅ / ❌ | [Cloudflare / Wordfence / none] |
| 2FA enabled for admins | ✅ / ❌ | |
| Activity/audit logging | ✅ / ❌ | |
| Automatic updates enabled | ✅ / ❌ | |

---

### 5. Final Verification
- Final malware scan: [CLEAN / issues — list]
- Core/plugin checksum verification: [PASSED / issues]
- mu-plugins / drop-ins / cron clean: [YES / NO]
- Site loads correctly + no cloaked redirect (Googlebot UA tested): [YES / NO]
- No PHP files in uploads/: [YES / NO]

---

### 6. Blacklist & Recovery Status
- Google Safe Browsing: [clean / removal requested DATE / still flagged]
- Sucuri SiteCheck / VirusTotal: [clean / flagged]
- Google Search Console review requested: [YES — DATE / NO / not needed]
- Estimated recovery time: [1–3 business days / already clean]

---

### 7. Monitoring & Maintenance Setup
- [ ] Security plugin active with email alerts
- [ ] WAF active
- [ ] Uptime monitoring: [UptimeRobot / other / not set]
- [ ] Automated off-site backups: [UpdraftPlus daily/weekly / other / not set]
- [ ] Google Search Console verified
- [ ] Reinfection re-scan scheduled: +24h / +72h / +7d / +30d

---

### 8. Root Cause & Recommendations
**Root cause:** [exactly how the attacker got in — be specific: plugin + version + CVE if known]

**Immediate recommendations:** 1. […] 2. […] 3. […]

**Long-term recommendations:** [managed hosting w/ WAF • staging env for testing updates • quarterly audits • minimize plugin count • drop nulled software for good]

---

### 9. Incident Timeline
| Time | Event |
|------|-------|
| [DATE] | Infection estimated to have started |
| [DATE] | Hack discovered |
| [DATE] | Remediation started / maintenance mode |
| [DATE] | Malware + persistence removed |
| [DATE] | Site back online |
| [DATE] | Google review requested |
| [DATE] | 30-day reinfection check |

---
*Report generated by Pulse AI WordPress Repair Skill — https://github.com/maomur/pulse-ai-wordpress-repair*

---

## Post-cleanup restore troubleshooting

After the owner re-uploads the cleaned site, these are the most common "the site is broken now" issues (all seen in real engagements):

- **Only sub-pages return `Not Found` / "The requested URL was not found", homepage OK** → the **`.htaccess` is missing**. It's a hidden dotfile and most FTP clients/file managers skip it by default. Re-upload it (or in wp-admin: **Settings → Permalinks → Save** to regenerate it). Also re-upload the hidden **`.user.ini`** (needed by AIOS).
- **Everything 404s, including the homepage** → **wrong upload nesting**: the files landed one level too deep (e.g. `…/docroot/www/…` instead of `…/docroot/…`). Move the *contents* of the inner folder up into the document root.
- **A custom/renamed login URL 404s** (e.g. AIOS "Rename Login Page" → `site.com/master`) → that URL is a **virtual** route created by the plugin via WordPress rewriting, so it **also depends on `.htaccess`**. Restoring `.htaccess` makes it work again. (The renamed login still uses the real `wp-admin/` folder — make sure that folder uploaded completely.)
- **Locked out of admin entirely (emergency recovery):** via File Manager/FTP, rename `wp-content/plugins/all-in-one-wp-security-and-firewall` (e.g. add `_OFF`). This deactivates AIOS and restores `/wp-login.php`; log in, re-save Permalinks, then rename the plugin folder back.
- **"Error establishing a database connection" after rotating the DB password** → update `DB_PASSWORD` (and if changed, `DB_USER`/`DB_NAME`/`DB_HOST`) in **`wp-config.php`** to match exactly; escape `'` and `\` in the password.

> Reminder to always give the owner: enable **"show hidden files"** so `.htaccess` and `.user.ini` actually upload.

---

## Constraints

### MUST (Required)
1. **Confirm authorization** before working on any site.
2. **Always make an off-server forensic backup** before any action — even of the infected site.
3. **Rotate all credentials AND secret keys/salts** — passwords alone don't kill stolen sessions.
4. **Hunt every persistence mechanism** — mu-plugins, drop-ins, cron, wp-config injection, auto_prepend_file, fake admins, app passwords. The job isn't done until these are clear.
4b. **Always run the database forensics pass** (Step 3B), live or on the provided `.sql` dump — a DB-only reinfector will rebuild file backdoors. Use serialization-aware tools (`wp search-replace`) for any edit inside serialized values; never a naive `REPLACE()`.
5. **Identify and close the entry point** (the vulnerable/nulled plugin) — updating files without closing the hole guarantees reinfection.
6. **Verify checksums** after reinstalling core and plugins.
7. **Test the site fully** (including a cloaked-redirect check with a Googlebot user-agent) before deactivating maintenance mode.
8. **Confirm clean** before requesting any blacklist removal.
9. **Schedule reinfection re-scans** (+24h/+72h/+7d/+30d); treat any return as a missed backdoor.
10. **Clean every site on the shared hosting account** together.
11. **Deliver the self-contained HTML dashboard** with real data in BOTH the simple and technical views; never leave `{{placeholders}}` unfilled, and never mark the status green if the entry point is still open.

### MUST NOT (Prohibited)
1. **Never delete files without a backup** — malware is evidence.
2. **Never assume an updated plugin is a cleaned plugin** — patches fix the hole, not the payload already injected.
3. **Never request Google review** while the site is still infected.
4. **Never reuse old passwords** after a compromise.
5. **Never keep unknown admin users or stray application passwords.**
6. **Never keep PHP files in wp-content/uploads/.**
7. **Never clean-and-keep nulled/pirated plugins or themes** — always replace with licensed copies.
8. **Never ignore the entry point** — no closed hole = certain reinfection.

---

## WP-CLI Quick Reference

```bash
# Core
wp core verify-checksums              # check core file integrity
wp core download --force              # reinstall core (keeps wp-content)
wp core update                        # update to latest version

# Integrity & scanning
wp plugin verify-checksums --all      # verify all plugin files vs wp.org
wp cron event list                    # list scheduled tasks (hunt malicious ones)
wp cron event delete <hook>           # remove a malicious cron event

# Database
wp db export backup.sql               # export database
wp db query "SELECT ..."              # run SQL query

# Users
wp user list --role=administrator     # list admins
wp user update ID --user_pass="..."   # reset password
wp user delete ID --reassign=1        # delete user
wp user session destroy --all --all-users   # force-logout everyone

# Config / secrets
wp config shuffle-salts               # rotate secret keys (invalidate stolen cookies)
wp option get users_can_register      # check open registration
wp option get default_role            # MUST be 'subscriber'

# Plugins & themes
wp plugin list --fields=name,status,version,update
wp plugin deactivate --all
wp plugin install slug --force        # reinstall clean
wp plugin update --all
wp theme delete slug

# Maintenance
wp maintenance-mode activate|deactivate
wp rewrite flush                      # regenerate clean .htaccess rules
wp cache flush
```

---

## References

- [WPScan Vulnerability Database](https://wpscan.com/) — search installed plugin/theme versions against known CVEs
- [Patchstack Database](https://patchstack.com/database/) — free WordPress vulnerability intelligence + virtual patching
- [Wordfence Threat Intel](https://www.wordfence.com/threat-intel/) & [Learning Center](https://www.wordfence.com/learn/) — CVE feed and free cleanup guides
- [Sucuri Blog](https://blog.sucuri.net/) — hack analysis and cleanup tutorials
- [WP-CLI Documentation](https://wp-cli.org/commands/) — full command reference (`wp search-replace`, `wp db`, `wp option`)
- [interconnect/it Search-Replace-DB](https://interconnectit.com/products/search-and-replace-for-wordpress-databases/) & [WP Search Replace CLI](https://github.com/andreekeberg/wp-search-replace-cli) — serialization-safe DB / dump editing
- [Google Search Console](https://search.google.com/search-console) — Security Issues & review requests
- [WordPress Security Guide (official)](https://developer.wordpress.org/advanced-administration/security/) — hardening docs
- [OWASP Top 10](https://owasp.org/www-project-top-ten/) — underlying vulnerability classes (XSS, Broken Access Control, etc.)

## Related Skills
- **security-best-practices** — general web security hardening
- **seo-geo** — recover SEO rankings after a pharma/SEO-spam hack
</content>
</invoke>
