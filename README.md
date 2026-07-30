# Seso H2A — H-2A Employer Prospecting Dashboard

A live, searchable prospecting tool built on the USCIS H-2A Employer Data Hub. Turns 152,236 raw petition records spanning FY2015–FY2026 into a ranked list of employers worth calling, with new-logo and growth signals.

**Live site:** enable GitHub Pages (Settings → Pages → Deploy from branch → `main` / root) and this URL becomes live: `https://<your-username>.github.io/Seso-H2A/`

## What it does

- **Search** any of 8,651 FY2026-active employers by name, city, or worksite state
- **Prospecting board**: top 500 employers ranked by a composite score — worker volume (35%), filing frequency (20%), multi-state complexity (10%), denial-rate pain (10%), and growth/new-logo signal (25%)
- **Growth & New Logos**: a 12-year trend of approved workers (FY2015–FY2026), a live list of 1,417 brand-new filers this fiscal year, and the fastest-growing existing employers (FY2024 → FY2025, both complete fiscal years)
- **Market overview**: approved workers by worksite state and industry (FY2026 snapshot)

## Data pipeline

Source: USCIS H-2A Employer Data Hub CSV export (UTF-16, tab-delimited), now covering all available fiscal years (2015–2026). Cleaning steps:
1. Convert UTF-16 → UTF-8
2. Strip thousands-separator commas from approval/denial count columns, cast to int
3. Sum all approval columns → `Total_Approved`; all denial columns → `Total_Denied`
4. Roll up to employer level, per fiscal year, to build a year-by-year history per employer
5. Flag **new filers**: first-ever petition on record filed in FY2026
6. Compute **YoY growth** using FY2024 → FY2025 only (both complete fiscal years) — FY2026 is a partial-year snapshot (through Q2) and is excluded from growth math to avoid an artificial "decline" from incomplete reporting
7. Growth/new-filer bonus in the scoring model only applies to employers with actual approved workers — a new filer with zero approvals (fully denied) gets no credit, since that's not a real account
8. Compute row-level totals separately (`totals.json`) — **do not** use employer/state rollups for headline totals, since `groupby` on a nullable key silently drops rows with a missing employer name or missing worksite state

## Folder structure

```
index.html              — the dashboard (static, no build step, no dependencies)
all_employers.json      — full search index, all 8,651 FY2026-active employers
top_employers.json      — top 500 ranked employers with scoring + growth fields
state_summary.json      — approved/denied workers by worksite state (FY2026)
industry_summary.json   — approved workers by industry (FY2026)
fiscal_year_trend.json  — approved workers by fiscal year, 2015–2026
new_logos.json          — top 50 brand-new FY2026 filers, ranked by scale
top_growers.json        — top 50 existing filers by FY2024→FY2025 growth
totals.json             — row-level true totals, FY2026 + all-time (see pipeline note above)
```

## Refresh pipeline (n8n)

Data refreshes quarterly when USCIS updates the Data Hub. An n8n workflow watches a Google Drive folder for the new quarterly export, re-runs the cleaning steps above, and commits updated JSON via the GitHub Contents API — GitHub Pages re-publishes automatically on push.

## Known limitations

- FY2026 is a partial-year snapshot (through Q2) — its bar and totals will look smaller than a complete fiscal year; this is a reporting-timing artifact, not a decline
- Some petitions have no employer name in the source file and are excluded from search/ranking (their worker counts still count toward the headline totals)
- Denial rates reflect USCIS's first adjudicative decision only, not appeals or revocations
