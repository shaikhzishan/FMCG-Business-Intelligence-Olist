# Project Documentation — FMCG Business Intelligence & Customer Analytics (Olist)

All figures in this document are sourced from the final executed notebook
(`executed_final.ipynb`, run on the real Olist dataset, zero errors across three
executions). For full-precision tables, see `ANALYTICAL_RESULTS.md` — this document
references it rather than duplicating every table in full.

## 1. Executive Summary

This project analyzes 99,441 Olist marketplace orders (Sept 2016 – Oct 2018) as a
retail/e-commerce BI case study: revenue trend, customer value and retention, category
and seller performance, delivery reliability, and a validated near-term revenue
forecast.

The most consequential finding is methodological rather than a business number: a
naive table join inflates revenue by 1.29x (R$20.3M vs. the corrected R$15.7M), because
two of the source tables (`order_items`, `payments`) have more than one row per order.
Every downstream metric in this project is built from a join structure that avoids
this, with assertions guarding the row counts.

On the business side: repeat purchase is rare (97% of customers buy once); delivery
delay has a statistically significant, meaningfully sized association with review
score (r = -0.336, 2.26 vs. 4.28 average stars for late vs. on-time); revenue and
seller value are concentrated (top 3 states = 63% of revenue, 18% of sellers = 80% of
revenue); and a validated forecast comparison found a simple seasonal-naive baseline
beat Prophet (43.0% MAPE vs. 88.1%) — a negative result for the more sophisticated
model, reported as found.

## 2. Business Context

Olist is a Brazilian marketplace that connects small and medium sellers to major
retail channels — it is not itself an FMCG company. This project treats the Olist
dataset as a stand-in for the kind of analytical questions a BI or analytics team asks
in retail/e-commerce/FMCG contexts generally: how is revenue trending, who are the
valuable customers, what's selling, is fulfillment reliable, and can near-term revenue
be forecast. The order/item/payment/review-level detail in this dataset is rich enough
to answer those questions properly, which is what makes it a useful case study
regardless of the specific industry label.

## 3. Business Questions

1. How has revenue moved over the observation window, and is that trend seasonal or
   structural (platform growth vs. organic demand)?
2. Which customers generate most of the value, and do they come back?
3. Which product categories and sellers actually drive revenue?
4. Is delivery performance related to how customers rate their orders?
5. Do certain product categories tend to be purchased together?
6. Given ~2 years of monthly data, can near-term revenue be forecast with any
   reliability, and does a more sophisticated model actually help?

## 4. Dataset

- **Source:** Olist public e-commerce dataset (Brazilian marketplace, 2016-2018).
- **Tables:** 9 CSVs — `customers`, `orders`, `order_items`, `order_payments`,
  `order_reviews`, `products`, `sellers`, `geolocation`, `product_category_name_translation`.
- **Scale:** 99,441 orders, 112,650 order items, 103,886 payment records, 100,000
  review records, 32,951 products, 3,095 sellers, 96,096 unique customers, 71 product
  categories.
- **Date range:** 2016-09-04 to 2018-10-17.
- **Relationships:** see `DATA_DICTIONARY.md` for the full key structure. The
  short version: `orders` is the spine (order-level grain); `order_items` and
  `order_payments` both have more rows than there are orders (1.14 and 1.04
  rows/order respectively), which is the structural fact behind the join-grain
  correction described in Section 5.
- **Known limitations:** no cost data, anonymized customer IDs, `customer_id` is
  order-specific (use `customer_unique_id` for person-level analysis) — full list in
  `LIMITATIONS.md`.

## 5. Data Preparation

**Loading:** all 9 CSVs read directly; date columns (`order_purchase_timestamp`,
`order_approved_at`, `order_delivered_carrier_date`, `order_delivered_customer_date`,
`order_estimated_delivery_date`, review timestamps) parsed to datetime.

**Missing values:** `order_approved_at` 0.2% missing, `order_delivered_carrier_date`
1.8%, `order_delivered_customer_date` 3.0% — all expected for orders that never
progressed to that stage (canceled, still in transit at data cutoff).

**Duplicates:** zero exact-duplicate rows in any of the 8 tables loaded.

**Grain check (the key diagnostic):** rows-per-`order_id` computed for `orders`
(1.00), `order_items` (1.14), `payments` (1.04), `reviews` (1.01). This confirms the
multi-row-per-order risk in two tables before any join is attempted.

**Derived fields:** `purchase_year_month`, `delivery_days`, `delivery_delay_days`,
`is_late`, and a `delivery_flag` for the 2 orders with implausible (negative or
>200-day) delivery times, which are excluded from delivery-time analysis specifically.

**Join strategy — two tables at two grains:**
- `order_facts`: one row per order (99,441 rows, assertion-checked). Payments are
  summed to the order level and reviews deduplicated to one per order *before*
  joining to `orders` — this is what prevents the row-multiplication bug.
- `item_level`: one row per order item (112,650 rows, assertion-checked). Used only
  for product/seller/basket analysis, never for order-level revenue totals.

Full detail on why this structure exists: `METHODOLOGY.md`, Revenue calculation
section.

## 6. Revenue Analytics

| Metric | Value |
|---|---:|
| Total revenue | R$ 15,739,137.01 |
| Valid orders | 98,207 |
| Average order value | R$ 160.26 |
| Median order value | R$ 105.28 |
| Cancellation/unavailable rate | 1% |

The mean sitting well above the median confirms a right-skewed order-value
distribution — a relatively small number of large orders pull the average up. This is
reported directly; no attempt is made to normalize it away.

**Naive vs. corrected revenue (the join-grain fix, quantified):** a naive chained
merge (`orders → order_items → payments`, summed) produces R$20,308,135 — a 1.29x
inflation over the corrected R$15,739,137. This was computed specifically to
demonstrate the size of the bug on this dataset, not left as a theoretical concern.

**Monthly trend and growth:** see Section 17 (the growth-calculation fix is
significant enough to warrant its own detailed writeup there) and
`ANALYTICAL_RESULTS.md` Section 5 for the full monthly table. Headline: 14.4% average
monthly growth across 20 stable months (2017-01 through 2018-08), with the top 3
customer states accounting for 63% of revenue.

## 7. Customer Analytics

**RFM base table:** built for 94,990 customers with at least one valid order (of
96,096 total unique customers). Mean recency 243.45 days, mean frequency 1.034
orders, mean monetary value R$165.69. The frequency statistic is the most important
one here — a mean of 1.034 orders per customer, with a median of exactly 1, confirms
that repeat purchase is the exception, not the norm, in this dataset.

**Customer Lifetime Value (historical, not predictive):** mean CLV R$165.69, median
R$107.90 — again right-skewed (max observed customer value R$13,664.08). Median is
used as the headline figure in all business-facing summaries.

**Cohort retention:** 23 monthly cohorts, sizes ranging from 1 to 7,190 customers.
Retention drops from 100% (month 0, by definition) to 5.2% (month 1) to under 1% from
month 2 onward (averaged over cohort-index values with at least 3 supporting cohorts,
to avoid a single small cohort skewing the average). This is built on deduplicated
order-level records specifically to avoid re-introducing the row-multiplication issue
into a repeat-purchase count.

**Customer geographic distribution:** revenue and customer counts both concentrate in
a small number of states (top 3 = 63% of revenue) — see Section 4/`DATA_DICTIONARY.md`
for the `customer_state` field and Section 11 (Logistics) for how this interacts with
delivery time.

## 8. Customer Segmentation

**RFM segments** (10 segments from a standard 25-cell R×F grid — not ad-hoc
thresholds): largest by customer count is Hibernating (15,252 customers, 16.1%),
largest by revenue share is Loyal (16.5% of revenue from 16.0% of customers). Full
segment table: `ANALYTICAL_RESULTS.md` Section 7.

**K-Means segmentation:** features (recency, frequency, monetary, average review
score) standardized before clustering; candidate K tested from 2 to 8; K selected by
silhouette score, which peaked at K=2 (silhouette 0.657). Resulting clusters:

| Cluster | Recency | Frequency | Monetary (R$) | Avg Review |
|---|---:|---:|---:|---:|
| 0 (single-purchase) | 243.98 | 1.00 | 160.52 | 4.10 |
| 1 (repeat/higher-value) | 226.57 | 2.11 | 330.07 | 4.14 |

Interpretation: the split is driven by frequency and spend, not satisfaction —
average review score is nearly identical between clusters (4.10 vs. 4.14). This rules
out "the low-value cluster is unhappy" as an explanation for the value gap.

## 9. Product and Category Analytics

Built from `item_level` (valid orders), so "revenue" here is item price + freight —
distinct from the order-level payment total used elsewhere. Top category by revenue:
health & beauty (R$1,437,665.78, 8,800 orders, 4.13 avg review). Full top-10 table and
all 71 categories' underlying data: `ANALYTICAL_RESULTS.md` Section 11.

Review score is attached at the order level and shared across items in a multi-item
order — a documented approximation for category-level review averages, not an exact
per-item figure.

## 10. Seller Analytics

3,053 of 3,095 sellers have at least one item in a valid order. Revenue is
concentrated: 557 sellers (18%) generate 80% of revenue. Top seller by revenue
(`4869f7a5dfa277a7dca6462dcf3b52b2`, R$249,393.44) has an above-average review score
(4.12); the 2nd-ranked seller by revenue (`7c67e1448b00f6e969d365cea6b010ab`,
R$239,536.44) has a below-average review score (3.34) — flagged as an account-level
risk in `BUSINESS_INSIGHTS.md`. Full seller table: `ANALYTICAL_RESULTS.md` Section 12.

## 11. Logistics and Delivery

| Metric | Value |
|---|---:|
| Delivered orders analyzed | 96,474 |
| Late rate (vs. estimated delivery date) | 6.8% |
| Average delivery time | 12.1 days |
| Median delivery time | 10 days |

Relationship with satisfaction (see Section 12 for full statistical detail): late
orders average 2.26 stars vs. 4.28 for on-time orders, a statistically significant
difference (Welch's t-test, t = -102.463, p < 0.001) with a moderate negative
correlation between delivery days and review score overall (r = -0.336, p < 0.001).
This is the single strongest, most statistically robust relationship found anywhere
in this project.

## 12. Statistical Analysis

| Test | Question | Method | Statistic | p-value | n | Result |
|---|---|---|---:|---:|---:|---|
| Delivery days vs. review score | Does longer delivery relate to lower reviews? | Pearson correlation | r = -0.336 | < 0.001 | 96,474 | Significant negative association |
| Late vs. on-time review score | Do late orders get worse reviews specifically? | Welch's t-test | t = -102.463 | < 0.001 | 96,474 | Significant; 2.26 vs. 4.28 average |
| Installments vs. order value | Do bigger orders use more installments? | Pearson correlation | r = 0.319 | < 0.001 | 99,441 | Significant positive association |

**Interpretation and limitation, stated once for all three tests:** with sample sizes
in the tens of thousands, p-values this small are expected even for modest effect
sizes — the correlation coefficients (r = -0.336, r = 0.319) are the more informative
numbers for judging practical relevance than the p-values alone. None of these tests
establish causation; all are reported as associations.

## 13. Payment Analytics

| Payment Type | Revenue Share | Orders |
|---|---:|---:|
| Credit card | 78.5% | 74,116 |
| Boleto | 18.0% | 19,539 |
| Voucher | 2.2% | 3,037 |
| Debit card | 1.3% | 1,514 |

Installment count correlates positively with order value (r = 0.319, p < 0.001),
consistent with installments being used for affordability on larger purchases rather
than representing anomalous behavior.

## 14. Association Rules

725 orders (2.2% of valid orders) span more than one product category and were used
as the transaction set for Apriori (`min_support=0.01`, 48 frequent itemsets found;
`min_lift=1.0`, 22 rules found). Strongest rule: perfumery ↔ health & beauty (lift
4.62, support 1.5%). Full rule table: `ANALYTICAL_RESULTS.md` Section 16.

**On interpreting lift correctly:** a lift of 4.62 means the pair co-occurs about
4.62x more often than independence would predict — it is a statement about aggregate
co-occurrence frequency, not a probability statement about any individual customer
("customers are 4.62x more likely to buy both" is an inaccurate paraphrase and is
avoided throughout this documentation).

Given the low absolute support (all 22 rules under 6%), these are presented as
cross-sell hypotheses to test, not established purchasing behavior.

## 15. Forecasting

- **Target:** monthly revenue from valid orders.
- **Stable months used:** 20 (of 24 total; 4 flagged as partial launch/tail months —
  see Section 17).
- **Split:** 17 training months, 3 held-out test months (2018-06, 2018-07, 2018-08).
- **Prophet configuration:** `yearly_seasonality=True, weekly_seasonality=False,
  daily_seasonality=False`.
- **Baseline:** seasonal-naive (same calendar month one year earlier, falling back to
  the last training value where unavailable).

| Model | MAE (R$) | MAPE |
|---|---:|---:|
| Prophet | 894,262.8 | 88.1% |
| Seasonal-naive | 438,837.6 | 43.0% |

**The seasonal-naive baseline outperformed Prophet.** This is presented as the primary
forecasting result, not softened. The likely cause: 17 training months (under 1.5
years) isn't enough for Prophet to fit a reliable yearly seasonal component, and it
appears to extrapolate the sharp 2017 growth trend into a test period where growth had
actually leveled off. The forward 6-month forecast (refit on all 20 stable months)
averages R$1,492,523 against a historical average of R$784,358 — a jump that is
consistent with this trend-extrapolation explanation and is flagged as such rather
than presented as a confident demand signal.

## 16. Model Validation

Every model in this project was validated against a held-out or cross-checked
standard before being trusted:
- **K-Means:** silhouette score computed across K=2-8, not assumed.
- **Association rules:** filtered to lift ≥ 1.0, confirming genuine positive
  co-occurrence rather than support-threshold noise.
- **Forecast:** trained/tested with a strict temporal split (no test-period data used
  in training) and benchmarked against a naive baseline rather than evaluated in
  isolation.

Full validation detail, including the join-grain and growth-calculation corrections,
is in `VALIDATION.md`.

## 17. Key Findings

- A naive join inflates revenue 1.29x (R$20.3M vs. R$15.7M) — fixed at the data-model
  level with assertions, not a one-off number correction.
- An initial growth calculation produced a meaningless 33,463% average by comparing
  against near-empty launch months; restricting the growth denominator to months at or
  above 10% of peak volume corrects this to 14.4%.
- 97% of customers buy once; cohort retention falls to 5.2% by month 1 and under 1%
  from month 2 onward.
- Delivery delay is significantly associated with lower review scores (r = -0.336,
  p < 0.001; 2.26 vs. 4.28 stars, late vs. on-time).
- Revenue and seller value are concentrated (top 3 states = 63% of revenue; 18% of
  sellers = 80% of revenue).
- K-Means finds only 2 natural customer clusters, separated by frequency/spend, not
  satisfaction.
- 22 real association rules exist among the 2.2% of orders spanning multiple
  categories, led by perfumery ↔ health & beauty (lift 4.62).
- A validated forecast comparison found the seasonal-naive baseline beat Prophet
  (43.0% vs. 88.1% MAPE) — a negative result for the more complex model, reported as
  found.

## 18. Business Recommendations

Each tied directly to a finding above — full detail with reasoning in
`BUSINESS_INSIGHTS.md`:

1. Prioritize delivery-time improvement in the states with the slowest average
   delivery and meaningful order volume — the delivery/satisfaction link is the most
   statistically robust relationship in this project.
2. Target first-to-second-order conversion rather than a loyalty program, given how
   rare repeat purchase actually is.
3. Cross-reference top-revenue sellers and categories against review score to flag
   high-revenue, underperforming accounts (concrete example: seller
   `7c67e1448b00f6e969d365cea6b010ab`, 2nd by revenue, 3.34 avg review).
4. Test the association-rule pairs as product-page cross-sell placements, as
   hypotheses given their low support — not an assumed win.
5. Use the seasonal-naive baseline for near-term revenue planning instead of Prophet,
   and re-run this comparison periodically as training history accumulates.

## 19. Limitations

Full list in `LIMITATIONS.md`. Headline items: no cost/margin data (all revenue
figures are payment value, not profit); anonymized customer IDs (no acquisition/
marketing linkage); observational, non-causal statistical relationships; limited
~2-year historical window constraining forecast reliability; low-support association
rules.

## 20. Future Work

Realistic, data-supported extensions only:
- **Category-level retention cuts:** `item_level` already has the category data needed
  to check whether retention differs by first-purchase category — not built out in
  this pass.
- **Seller scorecard:** the seller table already combines revenue, review score, and
  (via a join to `order_facts`) delivery time — formalizing this into a ranked,
  monitored scorecard is a natural next step, not a new data requirement.
- **Re-running the forecast comparison** every few months as more training data
  accumulates, to check whether Prophet eventually earns its complexity over the
  naive baseline.
- **A/B testing the association-rule cross-sell pairs** to move from "correlated" to
  "actually improves conversion."

Not planned, because the data doesn't support it: profitability analysis (no cost
data), predictive/forward-looking CLV or churn modeling (no acquisition-time
features), or any causal delivery/satisfaction estimate (no experimental design
available in this dataset).
