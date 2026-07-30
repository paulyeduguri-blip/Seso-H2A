# Seso H2A — H-2A Employer Prospecting Dashboard

A live, searchable prospecting tool built on the USCIS H-2A Employer Data Hub (FY2026 Q2 snapshot). Turns 13,438 raw petition records into a ranked list of employers worth calling.

**Live site:** enable GitHub Pages (Settings → Pages → Deploy from branch → `main` / root) and this URL becomes live: `https://<your-username>.github.io/Seso-H2A/`

## What it does

- **Search** any of 8,651 named employers by name or worksite state
- **Prospecting board**: top 500 employers ranked by a composite score (worker volume 45%, filing frequency 25%, multi-state complexity 15%, denial-rate pain 15%), filterable by state and minimum score
- **Market overview**: approved workers by worksite state and industry

## Data pipeline

Source file: USCIS H-2A Employer Data Hub CSV export (UTF-16, tab-delimited). Cleaning steps:
1. Convert UTF-16 → UTF-8
2. Strip thousands-separator commas from approval/denial count columns, cast to int
3. Sum all approval columns → `Total_Approved`; all denial columns → `Total_Denied`
4. Roll up to employer level for the prospecting board (`data/top_employers.json`, `data/all_employers.json`)
5. Roll up to state and industry level (`data/state_summary.json`, `data/industry_summary.json`)
6. Compute row-level totals separately (`data/totals.json`) — **do not** use the employer/state rollups for headline totals, since `groupby` on a nullable key silently drops rows with a missing employer name (118 records) or missing worksite state (66 records). `totals.json` is the only true source for the aggregate numbers shown at the top of the page.

## Folder structure

```
index.html              — the dashboard (static, no build step, no dependencies)
data/
  all_employers.json    — full search index, all 8,651 named employers
  top_employers.json    — top 500 ranked employers with scoring fields
  state_summary.json    — approved/denied workers by worksite state
  industry_summary.json — approved workers by industry
  totals.json            — row-level true totals (see pipeline note above)
```

## Refresh pipeline (n8n)

Data refreshes quarterly when USCIS updates the Data Hub. An n8n workflow watches a Google Drive folder for the new quarterly export, re-runs the cleaning steps above, and commits updated JSON to `data/` via the GitHub Contents API — GitHub Pages re-publishes automatically on push.

## Known limitations

- FY2026 Q2 is a single-quarter snapshot, not multi-year history — no growth/trend analysis yet
- 118 petitions have no employer name in the source file and are excluded from search/ranking (their worker counts still count toward the headline totals)
- Denial rates reflect USCIS's first adjudicative decision only, not appeals or revocations
