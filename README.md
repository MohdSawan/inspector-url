# Enhanced URL Inspector

A single-file Python tool for website reconnaissance and lightweight security inspection. It fetches and analyzes a target URL (or domain), extracts metadata and links, evaluates security headers and cookies, probes TLS certificates and protocol support, performs DNS/WHOIS lookups, optionally queries certificate transparency / VirusTotal / Shodan, can take page screenshots via Playwright, and can persist scan history to a local SQLite database.

Table of contents
- [Features](#features)
- [Quick start](#quick-start)
- [Installation](#installation)
- [Usage examples](#usage-examples)
- [CLI reference (flags)](#cli-reference-flags)
- [Output format & JSON schema](#output-format--json-schema)
- [Database (history) schema & queries](#database-history-schema--queries)
- [Integrations & API keys](#integrations--api-keys)
- [Screenshots (Playwright)](#screenshots-playwright)
- [Security, privacy & legal](#security-privacy--legal)
- [Limitations & behaviours](#limitations--behaviours)
- [Extending & contribution](#extending--contribution)
- [Troubleshooting](#troubleshooting)
- [License](#license)
- [Changelog (high level)](#changelog-high-level)
- [Contact / Support](#contact--support)

## Features
- HTTP(S) fetch with redirect handling and a spoofable User-Agent
- HTML parsing: title, meta description, generator, links, and scripts
- Internal vs external link classification
- Concurrent external link HEAD checks (with GET fallback)
- Simple framework/CMS fingerprinting heuristics (WordPress, Drupal, Joomla, Next.js, React, Angular)
- Security header checks (HSTS, CSP, X-Frame-Options, etc.) and CSP issue detection
- Cookie flag parsing (Secure, HttpOnly, SameSite)
- TLS certificate extraction (subject, issuer, validity, SANs, signature algorithm) via native ssl + cryptography
- Best-effort TLS protocol support tests (TLS 1.0/1.1/1.2/1.3 where available)
- DNS lookups (A, AAAA, MX, NS, TXT) using dnspython
- WHOIS lookup using python-whois
- IP geolocation (via ip-api.com)
- Optional crt.sh lookup for certificate transparency results
- Optional VirusTotal domain lookup (v3 endpoints)
- Optional Shodan host lookups
- Quick open-port checks for a list of ports (default: 80,443)
- Optional page screenshot using Playwright
- Save/load scan history to/from a local SQLite DB (`url_inspector.db`)
- Human-friendly (--pretty) and machine-readable JSON output

## Quick start
1. Save `enhanced_url_inspector.py` somewhere on your machine.
2. Install dependencies (see below).
3. Run a basic scan:
   ```bash
   python enhanced_url_inspector.py example.com --pretty
   ```
4. Run a deep scan with link checks and screenshot:
   ```bash
   python enhanced_url_inspector.py https://example.com --deep --check-links --screenshot --save-db --pretty
   ```

## Installation

Recommended Python versions: 3.8+

Install runtime dependencies:
```bash
pip install requests beautifulsoup4 dnspython tldextract python-whois cryptography
```

(Optional) If you want screenshots:
```bash
pip install playwright
playwright install
```
Note: `playwright install` downloads browser binaries and must be run once after installing the package.

Suggested (minimal) requirements.txt:
```
requests
beautifulsoup4
dnspython
tldextract
python-whois
cryptography
playwright  # optional, only if you plan to use screenshots
```

If you use a virtualenv:
```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

## Usage examples

Simple scan:
```bash
python enhanced_url_inspector.py example.com
```

Pretty + JSON + save to DB:
```bash
python enhanced_url_inspector.py example.com --pretty --save-db
```

Deep scan (WHOIS, geo, port checks), CRT.sh and VirusTotal:
```bash
python enhanced_url_inspector.py example.com --deep --crtsh --vt-api-key $VT_API_KEY
```

Check only crt.sh results:
```bash
python enhanced_url_inspector.py example.com --check-crtsh-only
```

Specify ports for quick port scan:
```bash
python enhanced_url_inspector.py example.com --deep --ports 80,443,22
```

Take a screenshot and save to a specific filename:
```bash
python enhanced_url_inspector.py example.com --screenshot --screenshot-out example.png
```

## CLI reference (flags)

- target (positional): URL or domain to inspect (e.g., `example.com` or `https://example.com`)
- --deep: Enable WHOIS, IP geolocation, open ports, and optional VT/Shodan/crt.sh lookups
- --check-links: Check external links found on page (concurrent HEAD requests)
- --concurrency N: Concurrency for link checks (default: 10)
- --link-timeout N: Timeout for link HEAD requests in seconds (default: 6)
- --vt-api-key KEY: VirusTotal API key (or set env var VT_API_KEY)
- --shodan-key KEY: Shodan API key (or set env var SHODAN_KEY)
- --crtsh: Query crt.sh for certificate transparency entries
- --screenshot: Take a screenshot of the page (Playwright required)
- --screenshot-out PATH: Screenshot output path (default: screenshot.png)
- --ports "80,443": Comma-separated ports for quick port checks during deep scan
- --timeout N: HTTP request timeout in seconds (default: 8)
- --pretty: Print a human-friendly summary in addition to JSON
- --save-db: Save scan report to local SQLite DB (url_inspector.db by default)
- --check-crtsh-only: Run a crt.sh query and print JSON results, then exit

Example:
```bash
python enhanced_url_inspector.py example.com --deep --check-links --pretty --save-db
```

## Output format & JSON schema

The tool always prints a JSON object (pretty-printed). When `--pretty` is passed it prints a human readable summary before the JSON.

Top-level report fields:
- target: original target string
- checked_at: UTC timestamp when scan ran
- http: object with HTTP fetch results
  - url: original target in request
  - final_url: final URL after redirects
  - status_code: HTTP status (int or null)
  - headers: dict of response headers
  - title, meta_description, meta_generator
  - links: list of raw hrefs extracted
  - scripts: list of script srcs
  - internal_links_sample, external_links_sample
  - num_links
  - robots_txt: robots.txt content or null
  - sitemap_xml: sitemap content or null
- cert: TLS certificate info (subject, issuer, not_before, not_after, san, serial_number, sig_alg) or null
- tls_versions: dict of TLS protocol names to booleans (best-effort)
- dns: dict with keys "A","AAAA","MX","NS","TXT" mapping to lists
- ip: resolved IPv4 address or null
- geo: IP geolocation object (from ip-api.com) or null
- whois: WHOIS results dict (raw, registrar, creation, expiration, emails) or null
- open_ports: dict of port -> boolean (true=open) (only with --deep)
- fingerprints: list of detected framework/server tags
- security_headers: dict including common headers and cookie analysis
  - cookie_flags: array of cookie metadata objects {cookie, secure, httponly, samesite, attrs}
- link_checks: dict mapping link -> {status, content_type} or error info (when --check-links)
- vt: VirusTotal domain result (raw JSON) or null (when provided)
- shodan: Shodan host info (raw JSON) or null (when provided)
- crtsh: crt.sh results (list of objects) or null (when requested)
- suggestions: list of human-readable suggestions generated from the findings
- screenshot: path to screenshot or null (when --screenshot)

Note: Some providers (VT, Shodan, crt.sh) return provider-specific JSON; the tool stores it as-is under vt/shodan/crtsh.

## Database (history) schema & queries

By default, saved DB file: `url_inspector.db`. When `--save-db` is used, each scan is appended to the `scans` table.

Schema:
```sql
CREATE TABLE IF NOT EXISTS scans (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  target TEXT,
  timestamp TEXT,
  report_json TEXT
);
```

Example queries:

- List last 10 scans:
  ```sql
  sqlite3 url_inspector.db "SELECT id, target, timestamp FROM scans ORDER BY id DESC LIMIT 10;"
  ```

- Dump a single report (id = 42):
  ```bash
  sqlite3 -json url_inspector.db "SELECT report_json FROM scans WHERE id=42;"
  ```

- Export recent reports to files:
  ```bash
  sqlite3 url_inspector.db "SELECT id, target, timestamp, report_json FROM scans WHERE target='example.com' ORDER BY id DESC LIMIT 5;" \
  | jq -r '.[] | .report_json' > report-42.json
  ```

## Integrations & API keys

- VirusTotal: v3 API endpoint `/api/v3/domains/{domain}` used. Provide via `--vt-api-key` or environment var `VT_API_KEY`.
- Shodan: host lookup via `https://api.shodan.io/shodan/host/{ip}?key=KEY`. Provide via `--shodan-key` or environment var `SHODAN_KEY`.
- crt.sh: no API key required; public JSON endpoint used (`https://crt.sh/?q=%25{domain}&output=json`).

Be mindful of rate limits and usage policies when using third-party APIs.

## Screenshots (Playwright)

- Requires `playwright` package and browser binaries:
  - Install package: `pip install playwright`
  - Install browsers: `playwright install`
- The script uses Playwright's synchronous API to:
  - Launch a headless Chromium browser
  - Navigate to the page (wait_until="networkidle")
  - Capture a full-page screenshot

If Playwright or browser binaries are missing, screenshot functionality gracefully returns null.

## Security, privacy & legal

- Only scan systems you own or have permission to test. Unauthorized scanning may violate laws or terms of service.
- Scans may reveal or store sensitive data: headers, robots / sitemap entries, linked URLs, WHOIS emails. Protect the `url_inspector.db` file and printed output.
- Using third-party APIs shares the queried domain/IP with those services — check their privacy policies.
- This tool is intended for passive reconnaissance and informational use, not for active exploitation.

## Limitations & behaviours

- HTML parsing uses BeautifulSoup (lenient). JavaScript-rendered content may not appear unless using screenshot (which runs a headless browser).
- Link checking uses HEAD requests primarily; some servers respond differently to HEAD and may block HEAD or redirect. The tool falls back to GET in that case.
- TLS version "tests" are best-effort and dependent on the local Python/SSL library capabilities. Results may vary across platforms.
- DNS/WHOIS/IP geolocation are performed via public resolvers/APIs; rate limits and network issues may affect outcomes.
- Some servers/CDNs will present rate-limiting, bot blocks, or anti-scraping measures and may not return full data.
- The script attempts to handle common exceptions and continues where possible, returning partial results rather than failing entirely.

## Extending & contribution

Ideas for improvements:
- Add more fingerprinting heuristics (e.g., specific plugin detection for WordPress, plugin versions).
- Integrate with specialized TLS scanners (e.g., sslyze) for fuller TLS analysis.
- Add an output mode for CSV or custom templates.
- Add an option to re-check saved links in the DB on demand.
- Add tests (unit and integration) and a CI workflow.

If you want, I can:
- Add a `requirements.txt`
- Add a GitHub Actions workflow for linting and basic tests
- Create a sample `CONTRIBUTING.md` and `SECURITY.md`

## Troubleshooting

- Missing core dependencies:
  - The script prints a helpful message at import time; run:
    ```bash
    pip install requests beautifulsoup4 dnspython tldextract python-whois cryptography
    ```
- Playwright screenshot failing:
  - Ensure you ran `playwright install` as root/owner of virtualenv and that necessary system libs are installed (see Playwright docs for Linux).
- Slow network calls:
  - Increase `--timeout` value or reduce concurrency for link checks `--concurrency`.
- crt.sh query returns no JSON:
  - crt.sh sometimes throttles requests; retry or use `--crtsh` sparingly.

## License

Use, modify and redistribute as you see fit. A suggested license: MIT. Example LICENSE content:

```
MIT License

Copyright (c) 2026 <Your Name>

Permission is hereby granted, free of charge, to any person obtaining a copy...
```

(Replace with desired license and author details.)

## Changelog (high level)
- 1.0.0 - Initial single-file implementation covering HTTP analysis, basic security checks, DNS/WHOIS, link checking, TLS probing, optional VT/Shodan/crt.sh, screenshots via Playwright, and SQLite history.

## Contact / Support
- If you'd like help integrating this into a repo, generating a `requirements.txt`, or adding CI (GitHub Actions), I can:
  - Create the README file in your repo
  - Add a minimal `requirements.txt`
  - Add a basic GitHub Actions workflow for linting

Enjoy — and remember to scan responsibly.
