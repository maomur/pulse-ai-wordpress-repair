---
name: pulse-ai-wordpress-repair
description: "Pulse AI WordPress Repair - Clean and repair a hacked or compromised WordPress site. Use when the user mentions: wordpress hacked, wordpress infected, wordpress malware, wordpress virus, wordpress repair, wordpress cleanup, wordpress blacklisted, wordpress redirect hack, wordpress pharma hack, wordpress phishing, wordpress defaced, wordpress backdoor, wordpress suspicious files, wordpress spam, malware wordpress, sitio wordpress hackeado, wordpress infectado, limpiar wordpress, reparar wordpress, wordpress comprometido. Guides through full remediation: assessment, backup, malware scan, cleanup, hardening, and monitoring."
metadata:
  tags: wordpress, security, malware, repair, cleanup, hack, backup, hardening, remediation
  platforms: Claude, ChatGPT, Gemini
  version: 1.0.0
---

# Pulse AI WordPress Repair

Complete workflow for cleaning and repairing a hacked or compromised WordPress site. Follow every step in order — skipping steps leads to re-infection.

---

## Quick Reference: Common Hack Types

| Hack Type | Symptoms | Entry Point |
|-----------|----------|-------------|
| **Redirect Hack** | Visitors redirected to spam/porn/pharma sites | Malicious JS in theme files or DB |
| **Pharma Hack** | Search engines show drug/pill keywords | SEO spam injected in DB or files |
| **Backdoor** | Attacker regains access after cleanup | Hidden PHP files, eval() code |
| **Defacement** | Homepage replaced with attacker's message | File write via vulnerable plugin |
| **Phishing** | Fake login pages served from your site | New files uploaded by attacker |
| **Malicious Admin** | Unknown admin users in wp-admin | Credential theft or brute force |
| **Cryptominer** | Server CPU spikes, slow site | Injected JS or PHP scripts |
| **Mailer Spam** | Server blacklisted for sending spam | PHP mailer script in uploads/ |

---

## Workflow

### Step 1: Initial Assessment

**Identify the hack type and scope:**

```bash
# Check Google Safe Browsing status
# Open in browser: https://transparencyreport.google.com/safe-browsing/search?url=yourdomain.com

# Check Sucuri SiteCheck (free)
# Open in browser: https://sitecheck.sucuri.net/results/yourdomain.com

# Check if site is blacklisted
# Open in browser: https://www.virustotal.com/gui/url/[base64-encoded-url]/detection

# View site source for visible injections
curl -sL "https://yourdomain.com" | grep -E "(eval|base64_decode|unescape|fromCharCode|document\.write)" | head -20

# Check for hidden iframes or external JS
curl -sL "https://yourdomain.com" | grep -E "(<iframe|<script src)" | head -20

# Check recent file modifications (last 10 days)
find /path/to/wordpress/ -type f -name "*.php" -mtime -10 | head -50
find /path/to/wordpress/ -type f -name "*.js" -mtime -10 | head -30
```

**Assess infection scope:**
- [ ] Homepage visually defaced?
- [ ] Redirects happening? (check on mobile, check incognito)
- [ ] Google showing warnings in search results?
- [ ] Hosting provider suspended the account?
- [ ] Unknown admin users in wp-admin?
- [ ] Spam emails being sent from server?

---

### Step 2: Backup & Isolation

**CRITICAL: Always backup before any changes.**

```bash
# === BACKUP VIA WP-CLI ===

# Backup the database
wp db export backup-$(date +%Y%m%d-%H%M%S).sql --allow-root

# Backup all files (tar + gzip)
tar -czf backup-files-$(date +%Y%m%d-%H%M%S).tar.gz /path/to/wordpress/

# === BACKUP VIA cPanel ===
# cPanel > Backup Wizard > Full Backup
# OR: cPanel > phpMyAdmin > Export database

# === PUT SITE IN MAINTENANCE MODE ===
wp maintenance-mode activate --allow-root

# OR create manually:
# Create /path/to/wordpress/.maintenance file
echo "<?php \$upgrading = time(); ?>" > /path/to/wordpress/.maintenance
```

**Change ALL passwords immediately:**

```bash
# Reset WordPress admin password
wp user update admin --user_pass="NewStrongPassword123!" --allow-root

# Reset ALL user passwords (recommended after hack)
wp user list --field=ID --allow-root | xargs -I {} wp user update {} --user_pass="TempPass$(date +%s)!" --allow-root

# Generate new WordPress secret keys
# Get new keys from: https://api.wordpress.org/secret-key/1.1/salt/
# Replace old keys in wp-config.php
```

**Also change immediately:**
- [ ] Hosting cPanel/FTP password
- [ ] Database password (and update wp-config.php)
- [ ] Email account passwords associated with the site

---

### Step 3: Scan for Malware

```bash
# === WP-CLI SCAN ===
# Check WordPress core integrity
wp core verify-checksums --allow-root

# Check plugins integrity
wp plugin verify-checksums --all --allow-root

# === FIND SUSPICIOUS FILES ===

# Find PHP files in uploads/ (should NOT contain PHP)
find /path/to/wordpress/wp-content/uploads/ -name "*.php" -o -name "*.phtml" -o -name "*.php5"

# Find files with common malware patterns
grep -rl "eval(base64_decode" /path/to/wordpress/ --include="*.php"
grep -rl "eval(gzinflate" /path/to/wordpress/ --include="*.php"
grep -rl "eval(str_rot13" /path/to/wordpress/ --include="*.php"
grep -rl "preg_replace.*\/e" /path/to/wordpress/ --include="*.php"
grep -rl "assert(\$_" /path/to/wordpress/ --include="*.php"
grep -rl "\$_POST\[.*\](\$_POST" /path/to/wordpress/ --include="*.php"
grep -rl "FilesMan\|c99shell\|r57shell\|WSO shell" /path/to/wordpress/ --include="*.php"

# Find hidden PHP shells (common names)
find /path/to/wordpress/ -name "wp-login.php" ! -path "*/wp-login.php"
find /path/to/wordpress/ -name "*.php" | xargs grep -l "FilesMan" 2>/dev/null
find /path/to/wordpress/ -name ".*.php" -o -name "...php" -o -name "wp-*.php" ! -path "*/wp-includes/*" ! -path "*/wp-admin/*"

# Find recently modified PHP files (outside of known update dates)
find /path/to/wordpress/ -type f -name "*.php" -newer /path/to/wordpress/wp-config.php | grep -v ".git"

# Check wp-config.php for additions
cat /path/to/wordpress/wp-config.php

# === SCAN DATABASE FOR MALWARE ===
# Check wp_options for malicious redirects
wp db query "SELECT option_name, option_value FROM wp_options WHERE option_value LIKE '%eval(%' OR option_value LIKE '%base64_decode%' OR option_value LIKE '%<script%';" --allow-root

# Check for pharma hack keywords in posts
wp db query "SELECT ID, post_title FROM wp_posts WHERE post_content LIKE '%viagra%' OR post_content LIKE '%cialis%' OR post_content LIKE '%casino%';" --allow-root

# Check for injected JS in siteurl or home options
wp option get siteurl --allow-root
wp option get home --allow-root
```

---

### Step 4: Remove Malware

```bash
# === CLEAN WORDPRESS CORE ===

# Reinstall WordPress core files (does NOT affect wp-content or wp-config.php)
wp core download --force --allow-root

# Verify core checksums after reinstall
wp core verify-checksums --allow-root

# === REMOVE MALICIOUS FILES ===

# Delete PHP files found in uploads (they should NOT be there)
find /path/to/wordpress/wp-content/uploads/ -name "*.php" -delete
find /path/to/wordpress/wp-content/uploads/ -name "*.phtml" -delete

# Remove any found shell files (replace with actual paths found in Step 3)
# rm /path/to/suspicious-file.php

# === CLEAN THE DATABASE ===

# Remove malicious content from wp_options (example: injected redirect)
wp db query "UPDATE wp_options SET option_value = REPLACE(option_value, '<script>malicious_code_here</script>', '') WHERE option_value LIKE '%malicious_code_here%';" --allow-root

# Remove spam content from posts
wp db query "UPDATE wp_posts SET post_content = REPLACE(post_content, 'malicious_content_here', '') WHERE post_content LIKE '%malicious_content_here%';" --allow-root

# Clean malicious code from wp_postmeta
wp db query "DELETE FROM wp_postmeta WHERE meta_value LIKE '%eval(base64_decode%';" --allow-root

# === REMOVE UNKNOWN/MALICIOUS USERS ===

# List all admin users
wp user list --role=administrator --allow-root

# Delete suspicious admin user (replace USER_ID with actual ID)
# wp user delete USER_ID --reassign=1 --allow-root

# === CLEAN .HTACCESS ===
# Check for malicious redirects in .htaccess
cat /path/to/wordpress/.htaccess

# Regenerate clean .htaccess
wp rewrite flush --allow-root
```

---

### Step 5: Plugin & Theme Audit

```bash
# === LIST ALL PLUGINS AND THEMES ===
wp plugin list --allow-root
wp theme list --allow-root

# === DEACTIVATE ALL PLUGINS (safe mode diagnosis) ===
wp plugin deactivate --all --allow-root

# === REINSTALL PLUGINS FROM OFFICIAL REPOSITORY ===
# For each legitimate plugin:
wp plugin install plugin-slug --force --allow-root
wp plugin activate plugin-slug --allow-root

# === REINSTALL ACTIVE THEME ===
# For themes from the official repository:
wp theme install theme-slug --force --allow-root

# === DELETE UNUSED THEMES ===
# WordPress default themes are safe to keep; delete others not in use
wp theme delete old-unused-theme --allow-root

# === CHECK FOR NULLED/PIRATED PLUGINS/THEMES ===
# Nulled software is the #1 source of backdoors
# Replace any nulled plugin/theme with legitimate licensed version

# === UPDATE EVERYTHING ===
wp core update --allow-root
wp plugin update --all --allow-root
wp theme update --all --allow-root
```

**Manual checks for each plugin/theme:**
- [ ] Is it from wordpress.org or a reputable developer?
- [ ] When was it last updated? (> 2 years = risk)
- [ ] Is it still actively maintained?
- [ ] Was it nulled/pirated? (delete immediately and replace)
- [ ] Check plugin files against known clean versions

---

### Step 6: Harden Security

```bash
# === FILE PERMISSIONS ===

# Directories: 755 (rwxr-xr-x)
find /path/to/wordpress/ -type d -exec chmod 755 {} \;

# Files: 644 (rw-r--r--)
find /path/to/wordpress/ -type f -exec chmod 644 {} \;

# wp-config.php: 600 (rw-------)
chmod 600 /path/to/wordpress/wp-config.php

# === WP-CONFIG.PHP HARDENING ===
# Add to wp-config.php:
```

```php
// Disable file editing via wp-admin
define('DISALLOW_FILE_EDIT', true);

// Disable plugin/theme installation (optional, strict)
define('DISALLOW_FILE_MODS', true);

// Force SSL for admin and login
define('FORCE_SSL_ADMIN', true);

// Limit post revisions
define('WP_POST_REVISIONS', 5);

// Disable debug in production
define('WP_DEBUG', false);
define('WP_DEBUG_LOG', false);
define('WP_DEBUG_DISPLAY', false);
```

```bash
# === .HTACCESS HARDENING ===
# Add to the TOP of .htaccess (before # BEGIN WordPress):
```

```apacheconf
# Protect wp-config.php
<files wp-config.php>
order allow,deny
deny from all
</files>

# Protect .htaccess itself
<files .htaccess>
order allow,deny
deny from all
</files>

# Disable directory browsing
Options -Indexes

# Block access to xmlrpc.php (unless you need it)
<files xmlrpc.php>
order allow,deny
deny from all
</files>

# Block PHP execution in uploads/
<Directory "/path/to/wordpress/wp-content/uploads/">
  <Files "*.php">
    Order Deny,Allow
    Deny from all
  </Files>
</Directory>

# Block suspicious query strings
RewriteEngine On
RewriteCond %{QUERY_STRING} (\<|%3C).*script.*(\>|%3E) [NC,OR]
RewriteCond %{QUERY_STRING} GLOBALS(=|\[|\%[0-9A-Z]{0,2}) [OR]
RewriteCond %{QUERY_STRING} _REQUEST(=|\[|\%[0-9A-Z]{0,2})
RewriteRule .* index.php [F,L]
```

```bash
# === GENERATE NEW SECRET KEYS ===
# Copy new keys from: https://api.wordpress.org/secret-key/1.1/salt/
# Replace the 8 define() lines for keys in wp-config.php

# === DISABLE XML-RPC VIA WP-CLI ===
# Add to functions.php or use a plugin like "Disable XML-RPC"
wp eval 'add_filter("xmlrpc_enabled", "__return_false");' --allow-root

# === INSTALL SECURITY PLUGIN ===
# Option A: Wordfence (comprehensive)
wp plugin install wordfence --activate --allow-root

# Option B: Sucuri Security (monitoring + WAF)
wp plugin install sucuri-scanner --activate --allow-root

# Option C: iThemes Security
wp plugin install better-wp-security --activate --allow-root
```

**Additional hardening checklist:**
- [ ] Enable 2FA for all admin accounts
- [ ] Change default `admin` username to something unique
- [ ] Use a Web Application Firewall (WAF) — Cloudflare free tier or Wordfence
- [ ] Limit login attempts (Wordfence or Login LockDown plugin)
- [ ] Add reCAPTCHA to login and comment forms
- [ ] Move wp-login.php to a custom URL (WPS Hide Login plugin)
- [ ] Enable automatic updates for core and plugins

---

### Step 7: Verify & Monitor

```bash
# === DEACTIVATE MAINTENANCE MODE ===
wp maintenance-mode deactivate --allow-root
# OR delete the .maintenance file:
rm /path/to/wordpress/.maintenance

# === FINAL VERIFICATION ===

# Rescan for malware
grep -rl "eval(base64_decode" /path/to/wordpress/ --include="*.php"
grep -rl "FilesMan\|c99shell\|r57shell" /path/to/wordpress/ --include="*.php"
find /path/to/wordpress/wp-content/uploads/ -name "*.php"

# Verify core integrity one more time
wp core verify-checksums --allow-root
wp plugin verify-checksums --all --allow-root

# Check site loads correctly
curl -sL "https://yourdomain.com" | grep -c "<html"

# Check for remaining redirect code
curl -sL "https://yourdomain.com" | grep -E "(eval|base64_decode|document\.write)"

# === REQUEST GOOGLE BLACKLIST REMOVAL ===
# ONLY do this after site is fully clean
# 1. Verify ownership in Google Search Console: https://search.google.com/search-console
# 2. Run Security Issues report in Search Console
# 3. Click "Request Review" after fixing all issues
# Timeline: 1-3 days for Google to review

# === SET UP ONGOING MONITORING ===
# Wordfence: Enable email alerts for file changes and logins
# Uptime monitoring: UptimeRobot (free) https://uptimerobot.com
# Google Search Console: check Security Issues weekly

# === SCHEDULE REGULAR BACKUPS ===
# UpdraftPlus plugin: automated daily/weekly backups to Google Drive or Dropbox
wp plugin install updraftplus --activate --allow-root
```

**IMPORTANT — Final Report Requirement:**

At the end of the remediation process, you MUST generate a complete, professional incident report documenting everything that was done during this session. Do not use a generic template — fill in every field with the actual data collected and actions taken. The report must be detailed enough that another technician could understand exactly what happened and what was done without needing to ask questions.

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

- **Attack type:** [redirect hack / pharma hack / backdoor / defacement / phishing / malicious admin / cryptominer / spam mailer / other]
- **Confirmed entry point:** [nulled plugin name / outdated plugin name and version / brute force / stolen credentials / hosting vulnerability / unknown]
- **Estimated infection date:** [date or "unknown — oldest modified file: filename (date)"]
- **Infection scope:**
  - Files affected: [exact list of files found with malicious code]
  - Database tables affected: [e.g., wp_options, wp_posts, wp_postmeta]
  - Malware type found: [e.g., eval(base64_decode), PHP shell, JS redirect injector, SEO spam]
- **Blacklist status at start:**
  - Google Safe Browsing: [clean / flagged]
  - Sucuri SiteCheck: [clean / flagged — list detections]
  - VirusTotal: [clean / X engines flagged]

---

### 3. Actions Taken

#### 3.1 Backup
- Backup created: [YES / NO]
- Backup method: [WP-CLI / cPanel / manual]
- Backup location: [path or storage]
- Database backup file: [filename]
- Files backup: [filename]

#### 3.2 Malicious Files Removed
List every file that was found infected and the action taken:

| File Path | Issue Found | Action |
|-----------|-------------|--------|
| [/path/to/file.php] | [eval(base64_decode...) / PHP shell / injected JS] | Deleted / Cleaned |

#### 3.3 Database Cleanup
List every database change made:

| Table | Field | Issue | Action |
|-------|-------|-------|--------|
| wp_options | option_value | Malicious redirect JS in siteurl | Removed injection |
| wp_posts | post_content | Pharma spam keywords in X posts | Cleaned content |

#### 3.4 WordPress Core
- Core reinstalled: [YES / NO]
- Version: [e.g., 6.5.3]
- Checksum verification result: [PASSED / FAILED — list any failed files]

#### 3.5 Users
- Unknown/malicious admin users removed: [list usernames and IDs]
- All admin passwords reset: [YES / NO]
- Users with suspicious activity: [list or "none found"]

#### 3.6 Plugins & Themes
- Plugins deactivated for diagnosis: [YES / NO]
- Plugins reinstalled from official source: [list]
- Plugins deleted (unused or nulled): [list]
- Themes deleted (unused): [list]
- Nulled/pirated software found: [YES — list / NO]
- All plugins updated: [YES / NO — list any that could not be updated]
- All themes updated: [YES / NO]

#### 3.7 Credentials Rotated
- [ ] WordPress admin password(s)
- [ ] Database password
- [ ] FTP / hosting password
- [ ] cPanel password
- [ ] WordPress secret keys (wp-config.php)
- [ ] Email account passwords

---

### 4. Security Hardening Applied

| Measure | Applied | Notes |
|---------|---------|-------|
| DISALLOW_FILE_EDIT in wp-config.php | ✅ / ❌ | |
| New WordPress secret keys | ✅ / ❌ | |
| File permissions corrected (755/644/600) | ✅ / ❌ | |
| .htaccess hardened | ✅ / ❌ | Rules added: [list] |
| XML-RPC disabled | ✅ / ❌ | |
| PHP execution blocked in uploads/ | ✅ / ❌ | |
| Login attempts limited | ✅ / ❌ | Plugin used: [name] |
| Security plugin installed/activated | ✅ / ❌ | Plugin: [Wordfence / Sucuri / other] |
| WAF (Web Application Firewall) | ✅ / ❌ | [Cloudflare / Wordfence / none] |
| wp-login.php URL changed | ✅ / ❌ | |
| 2FA enabled for admins | ✅ / ❌ | |
| Automatic updates enabled | ✅ / ❌ | |

---

### 5. Final Verification

- Final malware scan result: [CLEAN / issues found — list]
- Core checksum verification: [PASSED / issues — list]
- Site loads correctly: [YES / NO]
- No redirect code detected: [YES / NO]
- No PHP files in uploads/: [YES / NO]

---

### 6. Blacklist & Recovery Status

- Google Safe Browsing: [clean / removal requested on DATE / still flagged]
- Sucuri SiteCheck: [clean / still flagged]
- Google Search Console review requested: [YES — DATE / NO / not needed]
- Estimated recovery time from Google: [1–3 business days / already clean]

---

### 7. Monitoring & Maintenance Setup

- [ ] Security plugin active with email alerts
- [ ] Uptime monitoring configured: [UptimeRobot / other / not set]
- [ ] Automated backups scheduled: [UpdraftPlus daily/weekly / other / not set]
- [ ] Google Search Console verified and monitored
- [ ] Next recommended manual audit: [DATE — 30 days from now]

---

### 8. Root Cause & Recommendations

**Root cause:** [Describe exactly how the attacker got in — be specific]

**Immediate recommendations:**
1. [Most critical action the client must take]
2. [Second priority]
3. [Third priority]

**Long-term recommendations:**
- [e.g., Consider managed WordPress hosting with built-in WAF]
- [e.g., Implement a staging environment to test plugin updates before going live]
- [e.g., Schedule quarterly security audits]

---

### 9. Incident Timeline

| Time | Event |
|------|-------|
| [DATE] | Infection estimated to have started |
| [DATE] | Hack discovered |
| [DATE] | Remediation started |
| [DATE] | Site taken to maintenance mode |
| [DATE] | Malware removed |
| [DATE] | Site back online |
| [DATE] | Google review requested (if applicable) |

---
*Report generated by Pulse AI WordPress Repair Skill — https://github.com/maomur/pulse-ai-wordpress-repair*

---

## Constraints

### MUST (Required)
1. **Always backup before any action** — even before reading files when possible
2. **Rotate all credentials** — admin passwords, FTP, cPanel, DB, secret keys
3. **Verify checksums** after reinstalling core and plugins
4. **Test site fully** before deactivating maintenance mode
5. **Confirm site is clean** before requesting Google blacklist removal
6. **Replace nulled plugins/themes** with legitimate versions — never clean and keep them

### MUST NOT (Prohibited)
1. **Never delete files without a backup** — malware evidence may be needed
2. **Never request Google review** while site is still infected
3. **Never reuse old passwords** after a compromise
4. **Never keep unknown admin users** — even if they seem inactive
5. **Never keep PHP files in wp-content/uploads/** — they should not exist there
6. **Never ignore the entry point** — if you don't close the hole, re-infection is certain

---

## WP-CLI Quick Reference

```bash
# Core
wp core verify-checksums              # Check core file integrity
wp core download --force              # Reinstall core (keeps wp-content)
wp core update                        # Update to latest version

# Database
wp db export backup.sql               # Export database
wp db import backup.sql               # Import database
wp db query "SELECT ..."              # Run SQL query

# Users
wp user list --role=administrator     # List admin users
wp user update ID --user_pass="..."   # Reset password
wp user delete ID --reassign=1        # Delete user

# Plugins & Themes
wp plugin list                        # List all plugins
wp plugin deactivate --all            # Deactivate all plugins
wp plugin install slug --force        # Install/reinstall plugin
wp plugin update --all                # Update all plugins
wp theme list                         # List all themes
wp theme delete slug                  # Delete a theme

# Maintenance
wp maintenance-mode activate          # Enable maintenance mode
wp maintenance-mode deactivate        # Disable maintenance mode
wp rewrite flush                      # Regenerate .htaccess
wp cache flush                        # Flush object cache

# Options
wp option get siteurl                 # Get site URL
wp option get home                    # Get home URL
wp option update siteurl "https://..."  # Update site URL
```

---

## References

- [Wordfence Learning Center](https://www.wordfence.com/learn/) — Free malware removal guides
- [Sucuri Blog](https://blog.sucuri.net/) — Hack analysis and cleanup tutorials
- [WP-CLI Documentation](https://wp-cli.org/commands/) — Full command reference
- [Google Search Console](https://search.google.com/search-console) — Security Issues & review requests
- [WordPress Hardening Guide](https://developer.wordpress.org/apis/security/) — Official security docs
- [iThemes Security Checklist](https://ithemes.com/security/) — WordPress hardening checklist

## Related Skills

- **security-best-practices** — General web security hardening
- **seo-geo** — Recover SEO rankings after a hack (pharma/spam hack cleanup)
