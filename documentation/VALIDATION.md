# Validation

## Execution validation

The final notebook was executed **three complete end-to-end runs on the real Olist
dataset, with zero errors on all three**. The last two runs produced consistent results
(the third run included two targeted code fixes described below, which changed two
specific numbers in an expected, explainable direction — everything else matched).

Each run was a fresh kernel execution via `jupyter nbconvert --execute` (equivalent to
Restart Kernel → Run All), not a partial or cell-by-cell re-run. Verified by checking
every code cell's output for `error` cell types after each run — none found.

## Data validation

| Check | Result |
|---|---|
| Row counts match known Olist dataset totals | Yes — 99,441 orders, 112,650 order items, 103,886 payments, matching the publicly documented dataset |
| Duplicate rows (exact) | 0 in every loaded table |
| Missing values | Concentrated in delivery timestamps (0.2%–3.0%, expected for undelivered/canceled orders) and review free-text (not used in analysis) |
| Grain check (rows per `order_id`) | `orders` 1.00, `order_items` 1.14, `payments` 1.04, `reviews` 1.01 — confirms the join-multiplication risk before it's addressed |
| Referential integrity | `order_facts` asserted equal to `orders.order_id.nunique()` (99,441); `item_level` asserted equal to `order_items` row count (112,650) |
| Implausible values | 2 orders with negative or >200-day delivery time, flagged and excluded from delivery-time analysis; no negative prices, weights, or payment values found |

## Analytical validation

### Revenue reconciliation
A naive chained merge (`orders → order_items → payments`, summed) was computed
alongside the corrected order-level calculation specifically to quantify the
join-grain bug: naive revenue R$20,308,135 vs. corrected R$15,739,137 — a 1.29x
inflation, confirmed on the real dataset (not just in principle).

### Growth calculation correction
**Problem found:** the initial monthly growth calculation used `pct_change()` computed
before excluding partial/edge months. Because Olist's launch period includes months
with as few as 1-2 orders (Sept 2016: 2 orders; Dec 2016: 1 order), the first "stable"
month's growth rate was measured against a near-empty baseline, producing an average
monthly growth figure of approximately 33,463% — not economically meaningful.

**Fix applied:** months are now flagged as partial based on order volume relative to
the peak month (threshold: below 10% of peak), which correctly catches all four
affected months (2016-09, 2016-10, 2016-12, 2018-09), not just the literal first/last
row of the series. Growth is then recalculated **within** the stable-months-only
series, so no stable month's growth is ever measured against an excluded month.

**Result after fix:** 14.4% average monthly growth — a plausible, defensible figure for
a two-year-old marketplace.

**Why this is a correction, not data removal:** the excluded months are real
observations and remain visible in the monthly revenue chart (marked distinctly). They
are excluded only from the *growth-rate denominator*, because a month with one or two
orders is not a meaningful baseline for interpreting "normal" platform growth — this is
a modeling decision about which months are informative for a growth-rate calculation,
not a claim that those orders didn't happen.

### Association-rule validation
Rules were generated with `min_support=0.01` and filtered to `lift >= 1.0`. All 22
rules found have lift meaningfully above 1 (range: ~2.2 to ~4.6), confirming they
reflect genuine positive co-occurrence rather than threshold noise. Support is reported
alongside lift specifically so the low absolute frequency of these patterns (all under
6% support) isn't hidden by a high lift number alone.

### Segmentation validation
- RFM: quintile-based scores were spot-checked to confirm even distribution across R
  and F bins (as expected from `pd.qcut`), and the resulting 10 segments collectively
  account for all 94,990 scored customers with no unassigned "Other" bucket.
- K-Means: silhouette score was computed across the full candidate range (K=2 to K=8),
  not assumed — K=2 was the actual maximum (0.657), not a default choice.

## Statistical validation

| Test | Statistic | p-value | Sample size |
|---|---:|---:|---:|
| Delivery days vs. review score (Pearson) | r = -0.336 | < 0.001 | 96,474 |
| Late vs. on-time review score (Welch's t-test) | t = -102.463 | < 0.001 | 96,474 (6,533 late / 89,941 on-time) |
| Payment installments vs. order value (Pearson) | r = 0.319 | < 0.001 | 99,441 |

All three p-values are effectively zero given the sample sizes involved (tens of
thousands of orders); the correlation coefficients (r = -0.336, r = 0.319) are the more
meaningful numbers for judging the strength of each relationship, and both are reported
explicitly rather than substituted with "significant."

## Forecast validation

- **Train/test split:** last 3 of 20 stable months held out as a test set (17 training
  months); test months are 2018-06, 2018-07, 2018-08.
- **Baseline:** seasonal-naive (same calendar month one year prior, or last training
  value as fallback).
- **Candidate model:** Prophet, `yearly_seasonality=True, weekly_seasonality=False,
  daily_seasonality=False`.
- **Error metrics:** MAE and MAPE computed on the held-out test months for both models.
- **Result:** Prophet MAPE 88.1% vs. seasonal-naive MAPE 43.0% — the baseline won.
- **Final model choice:** seasonal-naive baseline is the notebook's stated
  recommendation for near-term planning, explicitly because it backtested better, not
  because Prophet is assumed inferior in general.

This negative result for Prophet is preserved in the documentation exactly as found —
it is not reframed as a success, and the 88.1% figure is stated directly rather than
paraphrased as "high error."

## What was NOT validated (explicitly out of scope)

- No causal identification strategy exists in this data for the delivery→satisfaction
  or category→satisfaction relationships — association only, stated as such throughout.
- No cost/margin data exists, so no profitability calculation was attempted or validated.
- No external benchmark (industry AOV, industry retention rate) was available to
  validate findings against — comparisons are internal to this dataset only.
