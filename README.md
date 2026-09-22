# Skopos

[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](https://opensource.org/licenses/MIT)

> **Early access** — Skopos is a CLI tool right now. A full web app with dashboard, CRM, and team features is in development. Looking for testers and feedback — open an issue.

Lead generation tool for appointment setters. Scrapes profiles from 8 platforms, enriches them with verified emails using SMTP verification and company domain detection, and exports to CSV.

![Skopos CLI](screenshot.png)

## Install

```bash
git clone https://github.com/yourusername/Skopos.git
cd Skopos
pip install -e .
cp .env.example .env
```

Or without installing: `pip install -r requirements.txt` and use `python -m skopos`.

## Usage

```bash
skopos
skopos --verbose   # enable debug logging
skopos --version
skopos --help
```

Interactive CLI for scraping profiles, enriching leads with contact info, and exporting to CSV.

## Layout

```
Skopos/
├── skopos/
│   ├── __init__.py          package version
│   ├── __main__.py          python -m skopos
│   ├── cli.py               interactive menu, export, settings
│   └── scrapers/
│       ├── __init__.py      re-exports one scrape_* function per platform
│       ├── instagram.py     tiktok.py     linkedin.py    github.py
│       ├── youtube.py       twitch.py     linktree.py    pinterest.py
│       ├── enrichment.py    email discovery + SMTP verification
│       ├── stealth.py       proxy and user-agent rotation
│       └── utils.py         shared email/phone/number parsing
├── pyproject.toml           packaging + the `skopos` command
├── requirements.txt         runtime dependencies
├── .env.example             copy to .env
├── proxies.example.txt      copy to proxies.txt
├── screenshot.png           the banner shot above
├── .gitignore
├── LICENSE
└── README.md
```

`.env`, `proxies.txt` and exported CSVs are read from and written to the directory you run `skopos` in.

### Every file

| File | What it does |
|------|--------------|
| `skopos/__init__.py` | Package docstring and `__version__`, the single place the version is declared. |
| `skopos/__main__.py` | Two lines so `python -m skopos` works without installing. |
| `skopos/cli.py` | The whole terminal UI: gradient banner, platform/tools menu, one `*_interactive()` per scraper, bulk scrape from file, CSV export, settings screen, background update check. Loads `.env` from the working directory. |
| `skopos/scrapers/__init__.py` | Re-exports each platform's scraper under a uniform `scrape_<platform>` name. |
| `skopos/scrapers/instagram.py` | Parses public profile HTML with mobile user agents. Several regex fallbacks for Instagram's shifting markup, retries with a fresh proxy on extraction failure. |
| `skopos/scrapers/tiktok.py` | Reads the `__UNIVERSAL_DATA_FOR_REHYDRATION__` JSON blob from the profile page. Reuses one httpx client with browser-like headers. |
| `skopos/scrapers/linkedin.py` | The only scraper needing auth. Calls the internal Voyager API with your `li_at` cookie; `validate_cookie()` catches an expired cookie before a run starts. |
| `skopos/scrapers/github.py` | Unauthenticated REST API (60 req/hour). Skips empty profiles — no repos, no bio, no activity. |
| `skopos/scrapers/youtube.py` | Parses `ytInitialData` out of the channel page. Handles both `@handle` and `/channel/` URLs. |
| `skopos/scrapers/twitch.py` | Public GraphQL endpoint. Pulls social links out of channel panels and detects live/offline. |
| `skopos/scrapers/linktree.py` | One scraper for four link-in-bio hosts: Linktree, Stan Store, Linkr, Bio.link. Splits the links it finds into socials, website and email. |
| `skopos/scrapers/pinterest.py` | Parses the embedded JSON in the profile page for follower and pin counts, bio, website. |
| `skopos/scrapers/enrichment.py` | `LeadEnricher` — the email-finding pipeline described below, plus lead scoring and a threaded `enrich_bulk()`. |
| `skopos/scrapers/stealth.py` | Everything that keeps scrapers from getting rate-limited: user-agent rotation, randomised delays, proxy selection from the three `SKOPOS_*` sources, and a retry decorator. |
| `skopos/scrapers/utils.py` | Three shared parsers: `extract_email()`, `extract_phone()`, `parse_abbreviated_number()` for turning `"11.5K"` into `11500`. |
| `pyproject.toml` | Package metadata and the `skopos` console command. Reads its dependency list from `requirements.txt` rather than duplicating it. |
| `requirements.txt` | The runtime dependencies, and the single source of truth for them. |
| `.env.example` | Template for `.env`: LinkedIn cookie, Hunter.io key, proxy and delay settings. |
| `proxies.example.txt` | Template for `proxies.txt` — one proxy URL per line. |
| `.gitignore` | Keeps `.env`, `proxies.txt`, exported CSVs and caches out of git. |
| `LICENSE` | MIT. |
| `screenshot.png` | The banner shot at the top of this file. |

## Supported Platforms

| Platform | Auth Required | What It Scrapes |
|----------|--------------|-----------------|
| Instagram | None | profile, bio, followers, email, phone, links |
| TikTok | None | profile, bio, followers, likes, email |
| LinkedIn | Session cookie | profile, headline, bio, email |
| GitHub | None | profile, bio, repos, email, website |
| YouTube | None | channel name, description, subscribers, email, links |
| Twitch | None | profile, bio, followers, partner/affiliate status, social links |
| Pinterest | None | profile, bio, followers, pins, website |
| Linktree | None | all link-in-bio platforms (Linktree, Stan, Linkr, Bio.link) |

## Features

- 8 platform scrapers with consistent output format
- Email enrichment with SMTP verification (no paid API needed)
- Company domain detection from headline/bio
- Email pattern detection (first.last@company.com patterns)
- Lead scoring (0-100)
- Website deep scraping for contact info
- Bio link scraping (Linktree, WhatsApp, tel: links)
- CSV export with enriched data
- Bulk scraping from username lists (CSV/TXT)
- Proxy rotation and user-agent rotation
- Configurable scrape delay and proxy settings
- Automatic update checker with version enforcement
- Debug logging with `--verbose` flag

## LinkedIn Setup

1. Log into LinkedIn in Chrome
2. Open DevTools (F12) > Application > Cookies > linkedin.com
3. Copy the value of `li_at`
4. Add to `.env`:

```
LINKEDIN_COOKIE=your_li_at_cookie_value
```

## Proxy Setup

```
# Single proxy
SKOPOS_PROXY=http://user:pass@host:port

# Rotating proxies from file (one per line)
SKOPOS_PROXY_FILE=proxies.txt

# Free anonymous proxies (no config needed)
SKOPOS_FREE_PROXY=true
```

Proxies are optional. All scrapers work without them. Twitch automatically falls back to a direct connection if the proxy fails.

## How Enrichment Works

After scraping, Skopos enriches leads with verified email addresses:

1. Extracts email/phone from bio text
2. Scrapes the lead's website, /contact, and /about pages
3. Detects the company from their headline (e.g. "CEO at CompanyX")
4. Finds the company domain via DNS MX lookup
5. Detects the email pattern from existing emails on the site
6. Generates email candidates (first.last@, first@, etc.)
7. Verifies each candidate against the mail server via SMTP
8. Scores confidence 0-100% based on source and verification

No paid API required. Optional Hunter.io support for additional coverage.

## Limitations

- Instagram may require retries depending on region/IP
- TikTok may serve CAPTCHAs depending on region or IP
- LinkedIn cookies expire periodically
- GitHub API is limited to 60 requests/hour without a token
- SMTP verification may be blocked by some mail servers
- Free proxies are unreliable

## License

MIT
