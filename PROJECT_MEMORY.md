# Overnight Mountings Scraper ? Project Handoff Memory

Last updated: 2026-09-10

## Scope and client decisions

There are two independent deliverables. Never mix their data, databases, files, services, timers, or download links.

1. **Original weekly family scrape**
   - Input: `Overnight_Unique_SKUs_For_Scraping.xlsx`.
   - Scope: 7,223 families only.
   - Output: update media columns in a *new copy* of the original master workbook; never overwrite client source Excel.
   - White/default, Yellow Gold, and Rose Gold images/videos must be captured separately.
   - Export links on VPS:
     - CSV: `http://199.71.214.83/overnight/overnight_latest.csv`
     - XLSX: `http://199.71.214.83/overnight/overnight_latest.xlsx`
   - VPS scheduling: `overnight-weekly.timer`, Sundays at 02:00 CDT. It is separate from full-site work.

2. **New paid full-site scrape**
   - New standalone workbook only; do not modify the weekly source/output.
   - Client scope: all available mounting/jewelry families and SKUs; exclude loose diamonds, services, builders, etc.
   - Account-authenticated prices required.
   - Capture valid configuration combinations: product variant, metal, White/Yellow/Rose color, diamond quality, and finish.
   - Finish rule: prefer `Complete`; use `Semi-Mount` only when Complete is unavailable.
   - Ring finger size: exactly `7`; non-rings omit finger size.
   - Client asked for all available configuration values, not every ring size.

## Security rules

- Never put client username/password, browser profile, cookies, `.env`, Excel sources, generated output, DB, or logs into Git.
- Credentials are stored only on VPS in `/etc/overnight-fullsite.env`, permissions `600`; do not print/read its values.
- Do not bypass CAPTCHA. A browser/manual review must be used if one is presented.
- Do not upload `.venv`, `data/`, `downloads/`, Excel files, or weekly systemd unit files when deploying full-site code.

## Repository architecture

- `src/overnight_scraper/media.py`: explicit, safe color-media whitelist.
  - Yellow images: `.alt`, `.side.alt`, `.set.alt`
  - Rose images: `.alt1`, `.side.alt1`, `.set.alt1`
  - Yellow/Rose videos only for `.video.yellow` / `.video.rose`
  - unrecognized color suffixes are reported, not silently assigned.
- `src/overnight_scraper/fullsite.py`: full-site data model, standalone XLSX exporter, request prototype.
- `src/overnight_scraper/browser.py`: authenticated Playwright product configuration capture.
  - Uses the site price endpoint `/api/calculate-price-due-date/` as authoritative price source.
  - Variant labels map to exact style SKU and canonical variant URL from `product-variants` JSON.
  - Avoids using display price timing as source of truth.
- `src/overnight_scraper/fullsite_store.py`: SQLite state queue and result rows.
- `src/overnight_scraper/fullsite_production.py`: resumable full-site production runner.
  - Browser login and browser-based catalogue discovery.
  - Uses its own `data/fullsite.db` and `downloads/fullsite/overnight_fullsite_latest.xlsx`.
- `src/overnight_scraper/cli.py`: adds `fullsite-run`, `fullsite-status`, `fullsite-export` while keeping old weekly commands intact.
- `deploy/overnight-fullsite.service`: independent systemd service.
- `deploy/overnight-fullsite.env.example`: placeholders only; no real secret.

## Local validation completed

- API-backed configuration checks passed for:
  - Ring `85132-1/4`, size 7, Complete, White/Yellow/Rose media, account price `$524.20` in test configuration.
  - Non-ring `84908-.015S4`, no finger size query, account price `$600.85` in test configuration.
- Full test suite passed locally: `13 passed`.
- Local `.venv` contains Playwright and pytest; `pyproject.toml` and `requirements.txt` declare `playwright>=1.58`.

## VPS state (as of 2026-09-10)

VPS project root: `/opt/overnight-scraper` (CentOS 8, root-managed).

Full-site deployment performed:
- Uploaded: full-site source files, `pyproject.toml`, `requirements.txt`, deployment unit/template.
- Installed `playwright` dependency and Chromium.
- Installed service: `/etc/systemd/system/overnight-fullsite.service`.
- Credentials file exists and is mode `600`: `/etc/overnight-fullsite.env`.
- Service enabled: `overnight-fullsite.service`.
- Initial plain-requests login failed; old errors remain in `logs/fullsite.out.log` and are historical.
- Corrected `fullsite_production.py` was uploaded; it now uses authenticated Chromium discovery.
- Current observed state: service active; catalogue discovered **1,371 products**; `complete=1`, `running=1`, `pending=1369`, `failed=0`, `rows=0` at the last check.
- Do not restart service merely because old log lines show previous `FullSiteClient` errors.

Current VPS monitoring:
```bash
cd /opt/overnight-scraper
source .venv/bin/activate
overnight-scraper fullsite-status --db data/fullsite.db
systemctl status overnight-fullsite.service --no-pager -l
journalctl -u overnight-fullsite.service -f
tail -n 100 logs/fullsite.out.log
```

Service control:
```bash
systemctl start overnight-fullsite.service
systemctl stop overnight-fullsite.service
systemctl restart overnight-fullsite.service   # only after code/service change
systemctl enable overnight-fullsite.service
```

Full-site export after results exist:
```bash
overnight-scraper fullsite-export --db data/fullsite.db --output downloads/fullsite/overnight_fullsite_latest.xlsx
```

## Operational cautions / outstanding work

- The server job is expected to continue if laptop/internet disconnects because systemd owns it.
- Current `fullsite-status` reports counts but does **not yet calculate an ETA**. Estimate should be based on completed product timestamps only after enough representative products complete; do not promise a precise ETA prematurely.
- First rough estimate based on initial rate was 3?5 days, possibly up to 7 days, but this is low confidence because configuration counts vary widely by product.
- Inspect why `rows=0` after the first completed product before claiming resulting full-site workbook is complete. It may be a product with no valid configuration controls, but must be confirmed.
- The full-site output is not yet published to the existing Nginx download URL. Add a separate full-site public file only after validating exported rows.
- The production job currently writes export after its loop completes; for a long-running full run, add periodic atomic snapshot export/publish in a future improvement.

## Git state

- Existing earlier commit: `8ac7bdb feat: add weekly download exports`.
- New full-site files and related updates were staged locally for commit.
- GitHub push was blocked by an automated security gate because it saw safe environment-variable references (`OVERNIGHT_PASSWORD`, placeholder values), not a proven credential.
- No commit or push was performed by that blocked command.
- Before pushing, re-run a non-disclosing staged-file secret audit; do not stage `/etc/overnight-fullsite.env`, local browser profile, or generated data.
