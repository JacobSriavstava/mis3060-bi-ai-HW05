# MIS3060 HW05: Wildcat Capital Star Schema

Jacob Srivastava · MIS3060 Business Intelligence with AI · Villanova University

This assignment models the eight clean Wildcat Capital tables as a star schema. It identifies keys, grain and measure additivity with Python, then builds and checks the model in Power BI Desktop.

## Repository contents

| File / folder | What it is |
|---|---|
| `Data/` | The eight clean Wildcat CSVs (5 dimensions, 3 facts) |
| `explore.py` | Profiles each table: rows, columns, data types, unique values and blanks |
| `star_design.py` | Tests primary and foreign keys (including which `dim_date` column the fact dates match), fact-table grain and month-end dates |
| `additivity.py` | Classifies every numeric column as additive, semi-additive, non-additive or not a measure, and writes `additivity.md` |
| `star_design.md` | Design write-up: keys table, grain, measure additivity, and design and business questions |
| `additivity.md` | Generated additivity tables (copied into `star_design.md`) |
| `powerbi/hw05.pbix` | Saved Power BI model |
| `powerbi/model_view.png` | Model view screenshot showing all 8 tables and 12 relationships |

## How to run

```bash
python3 -m venv .venv && source .venv/bin/activate
pip install pandas
python3 explore.py            # profile the tables
python3 star_design.py        # key and grain checks (add --write-md to regenerate the keys table)
python3 additivity.py         # additivity tables → additivity.md
```

The scripts read the CSVs from `Data/`.

## Model summary

- **Facts:** `fact_fees` (one fee per client per month), `fact_portfolio_snapshot` (one snapshot per client per month), `fact_transactions` (one row per transaction)
- **Dimensions:** `dim_client`, `dim_advisor`, `dim_security`, `dim_benchmark`, `dim_date` (marked as the date table on `date`)
- **12 relationships**, all many-to-one, single direction and active. Fact dates join to `dim_date[date]`, not `date_id`.

## Key design decisions

- **`dim_client[advisor_id]` → `dim_advisor` was not created.** Each fact table links to `dim_advisor` directly, so this link would create an ambiguous second path.
- **`dim_security` → `dim_benchmark` was kept** (one snowflaked branch). It is the only way transactions can reach a benchmark.
- **`dim_client[client_since]` is not linked to `dim_date`.** Most values fall before 2020, outside the date table.
- **Blank `fact_transactions[security_id]` is expected.** Those 101,528 rows are cash-only transactions (deposits, withdrawals, advisory fees).

## Checks performed

- All 8 primary keys are unique and never blank; all foreign keys match with 0 orphans.
- Power BI row counts match the CSVs: 25 / 8 / 2,500 / 1,827 / 500 / 122,197 / 129,547 / 298,572.
- The only `(Blank)` member is the cash rows under `dim_security`; an advisor slicer filters all three facts.

## Tools

Python (pandas), Power BI Desktop on the Apporto virtual desktop. Claude (Anthropic) assisted with the Python scripts and the walkthrough of the Power BI steps.
