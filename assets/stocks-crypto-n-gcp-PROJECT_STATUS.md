# Project Status — Crypto/Stocks/Macro Analytics API (GCP)

**Project**: `stocks-crypto-n-gcp`
**Last updated**: 2026-09-10

## Goal

Portfolio project, built specifically to signal GCP competency (Cloud Run, BigQuery, Cloud Scheduler, IAM/auth patterns) for job search purposes. Not a product — no real end users, no monetization intent. Public access (API endpoints, Looker dashboard) exists so it can be pointed to on a CV/portfolio, not because it's meant to be actively used by others.

## What's tracked, and why

- **Crypto — BTC, ETH, SOL**: the three most well-known cryptocurrencies, chosen for broad recognizability rather than any particular investment thesis.
- **Stocks — SPY, QQQ, DIA**: deliberately picked to span different market segments — SPY (broad market baseline), QQQ (tech/growth), DIA (blue-chip/defensive) — so volatility comparisons across them are more interesting than three similar large-caps would be.
- **Macro — FEDFUNDS, DGS10**: Fed funds rate (policy-set) and 10-Year Treasury yield (market-determined) — a deliberate pairing of "what the Fed is doing" vs. "what markets think about it."

Overall selection is arbitrary in the sense that no specific trading or research thesis is being tested — the point is to have a broad, varied, genuinely interesting dataset flowing through a real pipeline, not to produce investment insight.

## MVP Definition of Done

MVP = the current pipeline (all 3 sources live + automated + historically backfilled) **plus**:
1. A simple Looker Studio dashboard (scorecards + one time-series view)
2. Two FastAPI endpoints (`/api/market-snapshot`, `/api/volatility/{symbol}`)

Once those two things ship, this MVP is considered complete. Everything else in this document's "Out of Scope" section is v2+, not blocking.

---

## Done

### Infrastructure
- GCP project created (`stocks-crypto-n-gcp`), billing linked, $5 budget alert set
- APIs enabled: Cloud Functions, BigQuery, Cloud Scheduler, Cloud Run, Cloud Build
- BigQuery dataset `market_data` (us-central1), three tables: `prices`, `stocks`, `macro` — all partitioned on `fetched_at`

### Live data pipeline (all three sources automated)
- **CoinGecko** (crypto: BTC/ETH/SOL) → `prices` table. Cloud Run function deployed, IAM/OIDC-authenticated, on Cloud Scheduler (daily). Rename from placeholder `cloud-function-testing` → `fetch-prices-gecko` in progress (new service created, needs Scheduler job repointed and old service deleted).
- **FRED** (macro: FEDFUNDS, DGS10) → `macro` table. Live, authenticated, on Cloud Scheduler (daily).
- **Alpha Vantage** (stocks: SPY/QQQ/DIA) → `stocks` table. Live, authenticated, on Cloud Scheduler (daily). Note: free tier is 25 requests/day, 1 request/second — must stay on daily cadence, code respects this with a 1-second delay between calls.

### Historical backfill (one-time, local scripts — not deployed)
- **Alpha Vantage**: ~100 days backfilled for SPY/QQQ/DIA via `TIME_SERIES_DAILY`, loaded through a staging table (`stocks_staging`) and merged into `stocks`.
- **CoinGecko**: ~100 days backfilled for BTC/ETH/SOL via `market_chart`, trimmed to avoid overlap with already-live dates, loaded through staging (`prices_staging`) and merged into `prices`. Known gotcha documented: the current day comes back at finer-than-daily granularity regardless of the `days` parameter — handled with a dedupe-by-date step.
- **FRED**: ~100 days backfilled for FEDFUNDS/DGS10. Known issue: **not yet deduped/cleaned** — flagged as a to-do, deliberately deferred out of MVP scope. FEDFUNDS is a monthly series (expect ~3-4 points per 100 days), DGS10 is daily on business days — this is expected, not a bug.
- Backfill pattern used throughout: **fetch → CSV (for manual QA in a spreadsheet) → BigQuery staging table → verified merge into main table**. Repo has scripts for each stage per source under `scripts/history_fetch/` and `scripts/history_upload/`.

### Local dev environment
- `.env` file for local API keys (`ALPHA_VANTAGE_KEY`, `FRED_API_KEY`), loaded via `python-dotenv`. `.gitignore` in place, excludes `.env`, `__pycache__`, credentials files.
- `gcloud` CLI installed and authenticated (Application Default Credentials) for local BigQuery client access.
- GitHub repo initialized and pushed (`stocks-crypto-n-gcp`), README drafted covering architecture, setup steps, stack, and roadmap.

### Data cleanup
- Cleaned up duplicate rows in `stocks` table (from repeated manual testing) using BigQuery Time Travel to recover a bad delete.

---

## In Progress / Known Issues

*Note: all items below are tracked for awareness, not urgency — none are blocking MVP completion and none are being actively worked at the moment. Revisit post-MVP.*

- **CoinGecko service rename**: new `fetch-prices-gecko` service exists and is deployed; still need to (1) confirm it's fully working end-to-end, (2) repoint its Cloud Scheduler job to the new URL, (3) delete the old `cloud-function-testing` service.
- **CoinGecko API key**: CoinGecko now requires a free "Demo" API key (`x-cg-demo-api-key` header) even for the basic price endpoint — the live function currently works without one (possibly grandfathered), but this should be added soon before it potentially breaks without warning.
- **FRED historical data**: not yet deduped/validated in the `macro` table — tracked as a known issue, to be fixed post-MVP.
- **Alpha Vantage service naming/region inconsistency**: service is named `fetch-stocks-alpha-vantage` (with a hyphen, inconsistent with script name `fetch_stock_info_alphavantage.py`) and deployed in `europe-west1`, while the other two services are in `us-central1`. Cosmetic/consistency issue, not functionally broken.
- **Secrets stored as plain env vars**, not GCP Secret Manager — acceptable for a solo MVP, flagged as a "what I'd improve with more time" item for the README.

---

## Remaining (in order)

1. **Finish CoinGecko rename** — confirm new service works, repoint Scheduler, delete old service.
2. **Add CoinGecko API key** as an env var (same pattern as FRED/Alpha Vantage), update the header in the live function's code.
3. **Build the FastAPI layer** (Cloud Run):
   - Root landing route listing available endpoints (FastAPI's auto-generated `/docs` covers most of this for free)
   - `/api/market-snapshot` — latest crypto + stocks + macro combined into one JSON response, grouped by category
   - `/api/volatility/{symbol}?days=30` — std dev of daily returns (annualized), price range, simple trend direction, with an honest note if less data is available than requested; `days` defaults to 30, clamped to a sane range (e.g. 1–365)
4. **Build a Looker Studio dashboard** connected directly to BigQuery (native, free, no-auth) — scorecards + one time-series comparison + a takeaway text block, using current data only (crypto/stocks/macro). Publish it public, link goes on CV/portfolio. Purpose: demonstrate Looker/BI proficiency as a distinct skill from the API layer.
5. **Repo/documentation cleanup**:
   - Push remaining scripts (FRED fetch, Alpha Vantage fetch/backfill, CoinGecko backfill, staging/merge scripts) to the repo
   - Update README: mark roadmap items complete, refresh the architecture diagram to reflect all 3 sources (not just 1), add a "debugging notes" section (the Cloud Run revision/traffic-routing gotcha, the requirements.txt import-error incident, the CoinGecko current-day granularity issue)
   - Add an explicit short paragraph noting Terraform/IaC was deliberately skipped for this MVP (manual console setup instead), with a one-line rationale — framed as a considered scope decision, not an omission
6. **File a GitHub Issue** for the FRED dedup task, so it's tracked rather than forgotten.

---

## Explicitly Out of Scope for This MVP (backlog / v2)

- MERVAL, RIPTE, ARS/USD MEP scraping (Argentina-specific macro sources)
- Portfolio simulator endpoint, correlation matrix endpoint
- Full web UI
- Terraform / Infrastructure as Code
- GCP Secret Manager migration (currently plain env vars)
- "Indexed peso vs crypto vs inflation" dashboard view (depends on the Argentina sources above)
