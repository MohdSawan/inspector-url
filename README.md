# Enhanced URL Inspector

A richer URL inspection / recon tool for quick website reconnaissance, basic security checks, link validation, TLS probing, DNS/W WHOIS lookups, optional VirusTotal / Shodan integrations, and optional screenshots.

This single-file Python tool performs an HTTP(S) fetch, parses HTML for metadata and links, evaluates common security headers and cookies, probes TLS certificates and protocol support, performs DNS and WHOIS lookups, optionally checks links concurrently, optionally queries crt.sh / VirusTotal / Shodan, can take a page screenshot with Playwright, and can persist scan history to a local SQLite database.

- Script: `inspector.py`
- DB file (when saving): `url_inspector.db`

## Features

- HTTP(S) fetch with redirects and User-Agent control
- HTML parsing: title, meta description, generator, links, scripts
- Internal/external link classification and optional concurrent link HEAD checks
- Basic fingerprinting (WordPress, Drupal, Joomla, Next.js, React, Angular, server detection)
- Security header checks and CSP evaluation
- Cookie flag parsing (Secure, HttpOnly, SameSite)
- TLS certificate details and protocol support testing (TLS 1.0/1.1/1.2/1.3 best-effort)
- DNS lookups (A, AAAA, MX, NS, TXT)
- WHOIS and basic IP geolocation
- Optional: VirusTotal domain reports, Shodan host info, crt.sh certificate transparency search
- Optional page screenshot via Playwright
- Save/inspect scan history in local SQLite DB
- Human-friendly summary (`--pretty`) + JSON output

## Installation

1. Clone or download the repository (or save `enhanced_url_inspector.py` to your machine).

2. Install required Python packages:
   ```bash
   pip install requests beautifulsoup4 dnspython tldextract python-whois cryptography
