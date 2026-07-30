# Seso H2A — H-2A Employer Prospecting Dashboard

A live, searchable prospecting tool built on the USCIS H-2A Employer Data Hub. Turns 152,236 raw petition records spanning FY2015–FY2026 into a ranked list of employers worth calling, with new-logo and growth signals.

**Live site:** enable GitHub Pages (Settings → Pages → Deploy from branch → `main` / root) and this URL becomes live: `https://<your-username>.github.io/Seso-H2A/`

## What it does

- **Fiscal year filter**: browse any year from FY2015 through FY2026 — the board, search, hero stats, and market overview all update
- **Search** employers by name, city, or worksite state within the selected year
- **Prospecting board**: top 500 employers per year ranked by a composite score — worker volume (35%), filing frequency (20%), multi-state complexity (10%), denial-rate pain (10%), and growth/new-logo signal (25%, relative to the prior year)
- **Growth & New Logos**: a fixed 12-year trend of approved workers (FY2015–FY2026, independent of the year filter above), a live list of 1,417 brand-new filers in FY2026, and the fastest-growing existing employers (FY2024 → FY2025)
- **Market overview**: approved workers by worksite state and industry for the selected year — with a clear notice instead of an empty chart for FY2015–2017, where USCIS didn't collect that data yet

## Data pipeline

Source: USCIS H-2A Employer Data Hub CSV export (UTF-16, tab-delimited), covering all available fiscal years (2015–2026). Cleaning steps:
1. Convert UTF-16 → UTF-8
2. Strip thousands-separator commas from approval/denial count columns, cast to int
3. Sum all approval columns → `Total_Approved`; all denial columns → `Total_Denied`
4. Roll up to employer level, per fiscal year, to build a year-by-year history per employer
5. Flag **new filers**: first-ever petition on record filed in that fiscal year (not applicable to FY2015, the first year in the dataset)
6. Compute **YoY growth** relative to the prior fiscal year only when the employer has actual approved workers — a "new filer" with zero approvals (fully denied) gets no growth credit, since that's not a real account
7. Sanitize NaN values (missing employer state/city) before writing JSON — Python's `json.dump` emits non-standard `NaN` literals by default, which break strict JSON parsers in the browser; every export runs through a sanitizer that converts these to `null`
8. Compute row-level totals separately — **do not** use employer/state rollups for headline totals, since `groupby` on a nullable key silently drops rows with a missing employer name or missing worksite state

## Folder structure

```
index.html                        — the dashboard (static, no build step, no dependencies)
state_summary_by_year.json        — approved/denied workers by worksite state, all years bundled (~60KB)
industry_summary_by_year.json     — approved workers by industry, all years bundled (~16KB)
totals_by_year.json               — row-level true totals per year, all years bundled (~2KB)
fiscal_year_trend.json            — approved workers by fiscal year, 2015–2026 (fixed, not year-filtered)
new_logos.json                    — top 50 brand-new FY2026 filers (fixed, not year-filtered)
top_growers.json                  — top 50 existing filers by FY2024→FY2025 growth (fixed, not year-filtered)
by_year/
  top_employers_YYYY.json         — top 500 ranked board for that year (~150KB each, lazy-loaded)
  all_employers_YYYY.json         — full search index for that year (~1–3MB each, lazy-loaded)
```

**Why the split:** the board/search data for all 12 years combined would be ~20MB — too heavy to load upfront. Only the default year (FY2026) loads at page start; switching years fetches that year's two files on demand and caches them in memory, so re-visiting a year never re-fetches.

## Refresh pipeline (n8n)

Data refreshes quarterly when USCIS updates the Data Hub. An n8n workflow watches a Google Drive folder for the new quarterly export, re-runs the cleaning steps above (including the NaN sanitizer), and commits updated JSON via the GitHub Contents API — GitHub Pages re-publishes automatically on push.

## Known limitations

- Worksite state and industry weren't consistently collected before FY2018 (effectively 0% coverage in FY2015–2017) — the Market Overview section shows a notice instead of an empty chart for those years
- Some petitions have no employer name in the source file and are excluded from search/ranking (their worker counts still count toward the headline totals)
- Denial rates reflect USCIS's first adjudicative decision only, not appeals or revocations
- The Growth & New Logos section is intentionally independent of the year filter above it — it's always a FY2025→FY2026 view, since "who's new" and "who's growing" are inherently about the most recent transition, not whichever year you're currently browsing
