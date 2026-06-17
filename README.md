# Part 1 — D2C Customer Churn: Data Audit, EDA & Business Understanding

This repository is **Part 1 of 4** of the D2C Customer Churn Intelligence & Retention API
capstone. It covers the data audit, exploratory analysis, and business-framing work that
precedes any segmentation or modeling. It is fully self-contained and does not require any other
part of the capstone to be reviewed.

## What's in this repo

| File | Purpose |
|---|---|
| `eda_audit.ipynb` | Full, executed analysis notebook: data loading, schema/join checks, data-quality audit, EDA, and 5 evidence-backed churn-risk hypotheses |
| `data_quality_report.md` | Standalone write-up of every data-quality issue found (missing values, duplicates, outliers, date issues, leakage-risk columns) with treatment recommendations |
| `business_memo.md` | Business-facing memo on what to investigate before launching a retention campaign, grounded in specific findings (not generic churn advice) |
| `requirements.txt` | Python dependencies needed to run the notebook |
| `data/` | The 7 raw CSV files used throughout this part (copied here so the repo is runnable standalone) |
| `charts/` | 13 saved PNG charts produced by the notebook (also rendered inline in the notebook itself) |

## Dataset

The dataset is a synthetic D2C personal-care brand's customer/order/support/web/campaign data,
with a snapshot date of **2025-09-30** and a 60-day forward-looking churn target. Files used:

- `customers.csv` — 2,400 customer profiles
- `orders.csv` — ~10,000 order-line records (spans pre- and post-snapshot dates)
- `support_tickets.csv` — ~1,920 support interactions
- `web_events_snapshot.csv` — 30-day web/app activity as of the snapshot date
- `churn_labels.csv` — target variable + train/validation/test split
- `rfm_modeling_snapshot.csv` — pre-built feature table (used here mainly to cross-validate hypotheses; full feature engineering is independently redone in Part 3)
- `intervention_history.csv` — most recent retention campaign sent to each customer

All files are included under `data/` with relative paths used throughout the notebook, so no
manual download step is required to reproduce this work.

**Important:** `orders.csv` contains both pre-snapshot orders (safe to use) and post-snapshot
orders (dated after 2025-09-30, used only to construct the churn label). The notebook explicitly
filters these out before any behavioural aggregation — see Section 3.4 of the notebook and the
"Date Consistency" section of `data_quality_report.md`.

## How to run

1. Create a virtual environment (optional but recommended):
   ```bash
   python3 -m venv venv
   source venv/bin/activate   # on Windows: venv\Scripts\activate
   ```
2. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```
3. Launch Jupyter and open the notebook:
   ```bash
   jupyter notebook eda_audit.ipynb
   ```
4. Run all cells top to bottom (Cell → Run All). The notebook reads from `data/` using relative
   paths, so it will run correctly as long as you launch Jupyter from the repository root.

The notebook has already been executed once and saved with all outputs/charts visible, so you can
also simply open and read it without re-running anything.

## How this maps to the marking rubric

| Component | Where to find it |
|---|---|
| Correct loading, joining, schema understanding | `eda_audit.ipynb`, Sections 1–2 |
| Data-quality audit and treatment recommendations | `eda_audit.ipynb` Section 3, and `data_quality_report.md` in full |
| Meaningful EDA and visual analysis | `eda_audit.ipynb` Section 4 (8 charts covering demographics, orders, monetary behaviour, support, returns, web activity, campaigns, churn distribution) |
| Churn-risk hypotheses grounded in evidence | `eda_audit.ipynb` Section 5 (5 hypotheses, each with a chart/table, specific customer IDs, and counts) |
| Business memo clarity and prioritization | `business_memo.md` |

## Key findings (summary)

- Recency is the single strongest churn signal: churn rate rises from 8.8% (ordered in the last
  week) to 72.3% (60+ days since last order).
- Web/app session activity is an earlier warning sign than order recency — churn falls from
  66.3% (0 sessions/30d) to 20.9% (11+ sessions/30d).
- Support-ticket volume alone is *not* a reliable risk signal — frequent ticket-raisers are
  disproportionately active, frequent buyers. The real risk combination is inactivity **plus** a
  negative-sentiment ticket.
- The CRM team's existing manual priority labels are directionally strong (10% → 28% → 75% churn
  across low/medium/high) but miss 49 customers and over-flag 294 — see `business_memo.md` for
  the full breakdown and next steps.
- 12 duplicate order records and 7 extreme order-value outliers were identified and need
  treatment before any RFM or modeling work (see `data_quality_report.md`).

## Notes on academic integrity

Every chart, table, customer ID, and statistic in this repository was produced by executing the
notebook against the provided dataset — none of the findings are generic or templated. The
notebook can be re-run end-to-end to reproduce every number cited in `data_quality_report.md` and
`business_memo.md`.
