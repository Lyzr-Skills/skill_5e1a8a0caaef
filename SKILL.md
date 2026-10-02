---
name: weekly-kpi-excel
description: Builds the weekly KPI report as a formula-driven, branded Excel workbook (Summary scorecard with week-over-week, 4-week average, target status, a 13-week Trend tab with charts, raw Data and KPI Definitions). Use this skill whenever the user asks for the weekly KPI report, weekly metrics, a KPI scorecard or tracker in Excel/xlsx, or wants to refresh last week's KPI workbook with new numbers, even if they just say "update the weekly numbers" or "send me this week's KPIs in a spreadsheet".
---

# Weekly KPI Workbook

Produces the same workbook structure every week so readers always know where to look. The script handles layout and formulas deterministically; your job is getting the data right, validating it, and writing the commentary.

## Inputs

1. **KPI data** as a long-format CSV: `week_start,kpi_id,value` (one row per KPI per week, `week_start` as YYYY-MM-DD on the configured week start day). Percent KPIs are fractions: `0.052`, not `5.2`. If the user supplies data in another shape (wide table, export, query result), reshape it to this format first.
2. **KPI definitions** in `references/kpi-config.json`: names, groups, formats, direction (is higher better?), targets, owners, definitions. If it still has `"_status": "TEMPLATE"`, tell the user the KPIs are examples and ask for their real list before producing a report for anyone else.
3. **Brand tokens** (optional but expected): if the `brand-guidelines` skill is installed, pass its `references/brand-tokens.json` with `--brand`. Without it the workbook uses a neutral fallback palette. The builder reads the brand's optional `status`, `table`, `chart`, `kpi_value`, `slides` and `fonts` groups (tinted status badges, header style, target-line color, KPI figure weight, title background, semibold family and Office font fallback), so never hardcode these in the script.

Include at least 13 weeks of history when available so trends and the 4-week average are meaningful.

## Workflow

1. **Validate and compute stats first:**
   ```bash
   python scripts/kpi_data.py --data weekly.csv --config references/kpi-config.json > stats.json
   ```
   Read every warning. Missing current-week values, duplicate rows, misaligned week dates, and percents stored as whole numbers are the most common data problems. Fix them or tell the user; never paper over a gap.
2. **Sanity-check the numbers.** Look for anything implausible (a KPI that moved more than ~30% week over week, a value of exactly 0, a sudden format change). Flag these to the user as possible data issues rather than writing commentary that explains them.
3. **Write commentary** to `notes.json` using only the numbers in `stats.json`:
   ```json
   {"highlights": ["..."], "risks": ["..."], "actions": ["..."]}
   ```
   See "Writing commentary" below.
4. **Build:**
   ```bash
   python scripts/build_kpi_workbook.py --data weekly.csv --config references/kpi-config.json \
     --brand <brand-guidelines>/references/brand-tokens.json --notes notes.json \
     --out Weekly_KPI_<week_start>.xlsx
   ```
   Add `--week YYYY-MM-DD` to report on a week other than the latest.
5. **Recalculate and verify.** The workbook is formula-driven; openpyxl writes formulas without cached values. If the `xlsx` skill is available, run its `scripts/recalc.py` on the output and confirm zero errors. Then spot-check that Summary "This Week" values match `stats.json`. (Excel itself recalculates on open, since the file sets full-calc-on-load.)
6. **Brand audit:** if `brand-guidelines` is installed, run its `scripts/check_brand.py` on the file.
7. **Deliver** the file with a 2–3 sentence summary: how many KPIs are on track / at risk / off track, and the single most important movement.

## What the workbook contains

| Tab | Purpose |
|---|---|
| Summary | One row per KPI: this week, last week, WoW Δ and %, 4-week avg, target, vs target, status, owner. Scorecard counts. Highlights / risks / actions. The week in `B2` and the at-risk band in `F2` are blue input cells; changing them re-runs the whole report. |
| Trend | 13-week (configurable) table driven by `Summary!B2`, plus one line chart per KPI |
| Data | Raw long-format rows as an Excel Table. Next week, paste new rows here and the workbook updates. |
| Definitions | KPI definitions, owners, sources, and the status rule |

Status is direction-aware: for "lower is better" KPIs (churn, resolution time), being under target is On track. WoW changes are colored by whether the move is good or bad, not by sign.

## Writing commentary

Readers skim. Each bullet should be one sentence that names the KPI, the number, and why it matters.

- **Highlights:** 2–3 genuine wins, biggest first. "Trial conversion reached 18.2%, above the 18% target for the first time this quarter."
- **Risks:** every Off track KPI, plus any At risk KPI that worsened. State the gap to target.
- **Actions:** only actions the user told you about or that are clearly assigned. Never invent owners, dates, or root causes. If the cause of a movement is unknown, say "driver not yet identified" rather than guessing.
- Percent KPIs change in percentage points ("+1.2 pp"), not percent.

## Updating the KPI list

To add or change a KPI, edit `references/kpi-config.json` (id, name, group, format, direction, target, owner, definition, source). Supported formats: `currency`, `percent`, `number`, `decimal`, `hours`. Keep this file in sync with the copy in the `weekly-kpi-deck` skill so the deck and workbook always agree.

## Sample data

`assets/sample_weekly_kpis.csv` contains 13 weeks for the example KPIs, and `assets/sample_notes.json` shows example commentary. Use them to test or demo the skill.