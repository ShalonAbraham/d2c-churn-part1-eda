# Data Quality Report — D2C Churn Capstone, Part 1

**Snapshot date:** 2025-09-30 | **Files audited:** 7 | **Universe:** 2,400 customers

This report summarizes every data-quality issue identified while auditing the raw dataset in
`eda_audit.ipynb`, with exact counts, affected records, and a treatment recommendation for each.
All numbers below were produced by re-running the audit notebook against the data in `data/`.

---

## 1. Schema & Join Integrity

| Check | Result |
|---|---|
| Duplicate `customer_id` in `customers.csv` | 0 — all 2,400 customer IDs are unique |
| Orphan `customer_id` values in `orders.csv` | 0 |
| Orphan `customer_id` values in `support_tickets.csv` | 0 |
| Orphan `customer_id` values in `web_events_snapshot.csv` | 0 |
| Orphan `customer_id` values in `churn_labels.csv` | 0 |
| Orphan `customer_id` values in `rfm_modeling_snapshot.csv` | 0 |
| Orphan `customer_id` values in `intervention_history.csv` | 0 |
| Customers with zero pre-snapshot orders | 0 — every customer has at least one order before 2025-09-30 |
| Customers with zero support tickets | 1,153 of 2,400 (48.0%) |

**Interpretation:** All joins are clean. The 1,153 customers with no support tickets are not a
data error — the data dictionary explicitly notes that not every customer raises a ticket, and a
left join from `customers` correctly produces `NaN`/absent rows for them, which should be treated
as "0 tickets" rather than imputed or dropped.

---

## 2. Missing Values

| File | Column | Nulls | % of rows | Cause | Treatment |
|---|---|---|---|---|---|
| `customers.csv` | `loyalty_tier` | 1,386 | 57.8% | Customer not enrolled in loyalty programme | Treat as explicit category `"Not Enrolled"`, do not impute with mode |
| `customers.csv` | `skin_type` | 401 | 16.7% | Self-reported field left blank at signup | Treat as explicit category `"Not Provided"` |
| `orders.csv` | `rating` | 80 | 0.8% | Customer did not leave a satisfaction rating | Exclude (not zero-fill) when computing average ratings; zero-filling would artificially depress satisfaction scores |
| `rfm_modeling_snapshot.csv` | `loyalty_tier` | 1,386 | 57.8% | Same root cause as above (pass-through column) | Same treatment |

No other file in the package has missing values in any column.

---

## 3. Duplicate-Like Records

**Issue:** 12 rows in `orders.csv` have an `order_id` ending in `_DUP`.

**Investigation:** Each `_DUP` row was matched against its corresponding base `order_id` (same ID
with the suffix stripped). In all 12 cases, the `_DUP` row is a byte-for-byte match on
`customer_id`, `order_date`, `category`, `quantity`, `gross_amount`, and `discount_pct` — these
are exact duplicate order records, not legitimate repeat purchases on the same day.

**Affected customers:** `CUST00153`, `CUST00628`, `CUST00837`, `CUST00848`, `CUST00869`,
`CUST00875`, `CUST01140`, `CUST01335`, `CUST01601`, `CUST01621`, `CUST01820` (x2 — this customer
has two separate duplicate pairs).

**Impact if untreated:** Each of these 12 customers would have their `frequency_180d` (or
180-day order count more generally) and `monetary_180d` overstated by exactly one duplicate
order, which would bias their RFM segment assignment (Part 2) and any frequency/monetary-based
feature in the churn model (Part 3) toward looking more "active" than they really are.

**Treatment:** Drop all rows where `order_id` ends in `_DUP` before any aggregation. This was
applied in `eda_audit.ipynb` (`orders_clean = orders[~orders['order_id'].str.endswith('_DUP')]`)
prior to computing any order-behaviour statistics.

---

## 4. Outliers

**Issue:** `gross_amount` in `orders.csv` ranges from ₹149 to ₹24,789.38, far beyond the typical
order value.

**Investigation:**
- Using the standard IQR rule (upper fence = Q3 + 1.5×IQR ≈ ₹1,619), 536 orders sit above the
  fence — a large share of this is simply the natural right skew of a small subset of genuinely
  larger personal-care baskets, and is not itself alarming.
- A much smaller, more clearly anomalous group of 7 orders exceeds ₹3,000, topping out at
  ₹24,789.38, ₹22,719.45, and ₹15,957.48. These are concentrated in `Fragrance` and `Baby Care`.
  Their implied unit price (`gross_amount / quantity`) runs into the thousands of rupees, versus a
  dataset-wide median implied unit price under ₹500 — consistent with a data-entry or
  currency/decimal-placement error rather than a real luxury transaction.

**Treatment recommendation:**
- For the 7 extreme outliers (> ₹3,000): flag for manual review against source-system records
  where possible; if no correction is available, winsorize to a reasoned cap (e.g. the 99th
  percentile of `gross_amount`) before computing any monetary aggregate used for segmentation or
  modeling, so a handful of records don't distort `monetary_180d` comparisons across an entire
  segment.
- For the broader 536 orders between the IQR fence and ₹3,000: these likely represent genuine
  higher-value baskets (e.g. bundle purchases) and do not require correction, but should be kept
  in mind when choosing summary statistics — median and trimmed means are more robust than raw
  averages for `monetary_180d`-style features given this skew.

---

## 5. Date Consistency

**Issue:** `orders.csv` contains both pre- and post-snapshot orders, as documented in the data
dictionary.

**Findings:**
- `order_date` ranges from 2024-01-09 to 2025-11-29.
- 1,872 of 10,009 orders (18.7%) have `order_date > 2025-09-30` — these exist solely to support
  construction of the `churn_next_60d` label and must never be used as model input features.
- Cross-checked every order against the corresponding customer's `signup_date`: zero orders were
  found to predate the customer's own signup, so there is no impossible-date issue to remediate.

**Treatment:** Any feature-engineering step performed on raw `orders.csv` (in this notebook, and
again independently in Part 2 and Part 3) must explicitly filter to `order_date <= '2025-09-30'`
before aggregating recency, frequency, or monetary signals. This filter was applied throughout
`eda_audit.ipynb`.

---

## 6. Leakage-Risk Columns

| Column / Rows | File | Risk | Treatment |
|---|---|---|---|
| Rows with `order_date > 2025-09-30` | `orders.csv` | Directly encodes post-snapshot purchasing behaviour | Exclude from all feature engineering; usable only to construct/verify the churn label |
| `churn_next_60d` | `churn_labels.csv`, `rfm_modeling_snapshot.csv` | Target variable | Never use as a model input feature |
| `split` | `churn_labels.csv`, `rfm_modeling_snapshot.csv` | Not leakage itself, but must be respected consistently | Use the same train/validation/test assignment across Parts 3 and 4 to keep evaluation honest |

---

## 7. Summary of Treatment Decisions Applied in This Audit

1. Dropped all 12 `_DUP` order rows before any order-behaviour aggregation.
2. Filtered `orders.csv` to `order_date <= 2025-09-30` before computing any pre-snapshot
   behavioural statistic.
3. Treated `loyalty_tier` and `skin_type` nulls as explicit "not enrolled" / "not provided"
   categories rather than imputing.
4. Excluded null `rating` values (rather than zero-filling) from average-rating calculations.
5. Flagged 7 extreme `gross_amount` outliers (> ₹3,000) for manual review / winsorization rather
   than blanket removal, since some high-value orders may be legitimate.
6. Confirmed zero orphan keys and zero impossible pre-signup orders — no remediation needed for
   join integrity or signup-date consistency.

These decisions are carried forward into the RFM segmentation (Part 2) and churn-model feature
engineering (Part 3) to ensure consistent, leakage-free, and bias-aware treatment of the same
underlying issues across the full capstone.
