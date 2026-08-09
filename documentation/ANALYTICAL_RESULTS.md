# Analytical Results Ledger

Every number in this file comes from the final executed notebook (`executed_final.ipynb`,
run against the real Olist dataset, zero errors, three consecutive executions with
consistent results across the final two). Where the notebook printed a rounded figure,
the same rounding is kept here so the two can be reconciled directly. This file is the
single source of truth for numbers used elsewhere in the documentation — if any other
document disagrees with this one, this one is correct.

## 1. Dataset

| Metric | Final Value | Interpretation |
|---|---:|---|
| Orders | 99,441 | Full order table, all statuses |
| Order items | 112,650 | 1.14 rows per order on average |
| Payment records | 103,886 | 1.04 rows per order on average |
| Review records (raw) | 100,000 | Before dedup to one review per order |
| Products | 32,951 | |
| Sellers (total) | 3,095 | 3,053 have at least one item in a valid (non-canceled) order |
| Unique customers | 96,096 | By `customer_unique_id` |
| Product categories | 71 | Per `product_category_name_translation`; all 71 appear in valid-order item data |
| Date range | 2016-09-04 to 2018-10-17 | Order purchase timestamps |

## 2. Data Quality

| Metric | Final Value | Interpretation |
|---|---:|---|
| `order_approved_at` missing | 0.2% | Expected — some orders never get approved (canceled) |
| `order_delivered_carrier_date` missing | 1.8% | Orders not yet/never shipped |
| `order_delivered_customer_date` missing | 3.0% | Orders not yet/never delivered |
| Duplicate rows, all tables | 0 | No exact-duplicate rows in any of the 8 loaded tables |
| `order_items` rows per order | 1.14 | Confirms multi-row-per-order risk |
| `payments` rows per order | 1.04 | Confirms multi-row-per-order risk (installments) |
| `reviews` rows per order | 1.01 | Small but non-zero — deduplicated to 1/order before use |
| Implausible delivery times flagged | 2 orders | Negative or >200-day delivery time; excluded from delivery-time analysis |

## 3. Revenue / Join-Grain Correction

| Metric | Final Value | Interpretation |
|---|---:|---|
| Naive chained-merge revenue | R$ 20,308,135 | `orders → order_items → payments`, summed `payment_value` — inflated |
| Corrected order-level revenue | R$ 15,739,137 | From `order_facts`, payments pre-aggregated to order grain |
| Inflation factor | 1.29x | Naive ÷ corrected |
| `order_facts` row count | 99,441 | Matches `orders.order_id.nunique()` exactly (assertion-checked) |
| `item_level` row count | 112,650 | Matches `order_items` row count exactly (assertion-checked) |

## 4. Business KPIs

| Metric | Final Value | Interpretation |
|---|---:|---|
| Total revenue | R$ 15,739,137.01 | Valid orders only |
| Total orders | 99,441 | All statuses |
| Valid orders | 98,207 | Excludes `canceled` / `unavailable` |
| Unique customers | 96,096 | |
| Average order value | R$ 160.26 | Right-skewed — see median |
| Median order value | R$ 105.28 | More representative "typical order" |
| Average items per order | 1.14 | |
| Average review score | 4.07 / 5 | |
| Cancellation / unavailable rate | 1% | Of all orders |

## 5. Sales & Growth

| Metric | Final Value | Interpretation |
|---|---:|---|
| Total calendar months in data | 24 | Sept 2016 – Oct 2018 |
| Volume threshold for "partial" month | 742 orders | 10% of the peak month (7,423 orders, Nov 2017) |
| Months flagged partial | 4 | 2016-09 (2 orders), 2016-10 (293), 2016-12 (1), 2018-09 (1) |
| Stable months used for growth/forecast | 20 | 2017-01 through 2018-08 |
| Best stable month | 2017-11 | R$ 1,172,639 |
| **Corrected** average monthly growth | **14.4%** | Computed within the 20 stable months only |
| **Original (buggy)** average monthly growth | ~33,463% | Computed before excluding near-empty launch months — not used in final notebook, documented for transparency |
| Top 3 states, share of revenue | 63% | Customer state |

## 6. Customer Metrics (RFM base table)

Built on 94,990 customers with at least one valid order (of 96,096 total unique
customers — the remainder had only canceled/unavailable orders).

| Metric | Mean | Std | Min | 25% | Median | 75% | Max |
|---|---:|---:|---:|---:|---:|---:|---:|
| Recency (days) | 243.45 | 153.00 | 1 | 119 | 224 | 352 | 729 |
| Frequency (orders) | 1.034 | 0.211 | 1 | 1 | 1 | 1 | 16 |
| Monetary (R$) | 165.69 | 226.74 | 0.00 | 63.10 | 107.90 | 182.94 | 13,664.08 |

## 7. RFM Segmentation

| Segment | Customers | Customer Share | Revenue (R$) | Revenue Share |
|---|---:|---:|---:|---:|
| Hibernating | 15,252 | 16.1% | 2,487,645.5 | 15.8% |
| At Risk | 15,232 | 16.0% | 2,452,407.0 | 15.6% |
| Potential Loyalist | 15,230 | 16.0% | 2,515,141.2 | 16.0% |
| Loyal | 15,217 | 16.0% | 2,596,443.8 | 16.5% |
| Champion | 7,694 | 8.1% | 1,363,148.1 | 8.7% |
| About to Sleep | 7,506 | 7.9% | 1,160,542.8 | 7.4% |
| Cannot Lose | 7,434 | 7.8% | 1,348,922.7 | 8.6% |
| Need Attention | 3,819 | 4.0% | 568,040.2 | 3.6% |
| New | 3,804 | 4.0% | 616,253.4 | 3.9% |
| Promising | 3,802 | 4.0% | 630,592.4 | 4.0% |

## 8. Customer Lifetime Value (historical)

| Metric | Final Value | Interpretation |
|---|---:|---|
| Mean CLV | R$ 165.69 | Pulled up by a right tail (max R$ 13,664.08) |
| Median CLV | R$ 107.90 | Headline figure — more representative |

## 9. Cohort Retention

| Metric | Final Value | Interpretation |
|---|---:|---|
| Number of monthly cohorts | 23 | |
| Cohort size range | 1 – 7,190 customers | |
| Month 0 retention | 100.0% | By definition |
| Month 1 retention | 5.2% | First real repeat-purchase signal |
| Month 2 retention | 0.3% | |
| Months 3–17 retention | 0.1% – 0.3% | Averaged over cohorts with ≥3 supporting cohorts |

## 10. K-Means Customer Segmentation

| Metric | Final Value | Interpretation |
|---|---:|---|
| Features used | recency, frequency, monetary, avg_review | Standardized (z-score) |
| Candidate K range tested | 2–8 | |
| Selected K | 2 | Highest silhouette score |
| Silhouette score at K=2 | 0.657 | |

| Cluster | Recency | Frequency | Monetary (R$) | Avg Review |
|---|---:|---:|---:|---:|
| 0 | 243.98 | 1.00 | 160.52 | 4.10 |
| 1 | 226.57 | 2.11 | 330.07 | 4.14 |

Interpretation: the split is driven almost entirely by frequency/spend, not
satisfaction — average review score is nearly identical across both clusters.

## 11. Product & Category Analysis (top 10 by revenue)

| Category | Revenue (R$) | Orders | Avg Price (R$) | Avg Review |
|---|---:|---:|---:|---:|
| health_beauty | 1,437,665.78 | 8,800 | 130.34 | 4.13 |
| watches_gifts | 1,298,292.47 | 5,604 | 200.70 | 4.01 |
| bed_bath_table | 1,240,386.13 | 9,399 | 93.36 | 3.88 |
| sports_leisure | 1,147,244.63 | 7,673 | 114.06 | 4.11 |
| computers_accessories | 1,050,941.58 | 6,654 | 116.22 | 3.94 |
| furniture_decor | 899,626.04 | 6,425 | 87.67 | 3.90 |
| housewares | 772,035.14 | 5,847 | 90.65 | 4.06 |
| cool_stuff | 704,086.24 | 3,616 | 164.27 | 4.14 |
| auto | 678,606.64 | 3,872 | 139.53 | 4.05 |
| garden_tools | 579,525.20 | 3,505 | 111.14 | 4.04 |

Total categories with valid-order revenue: 71 of 71.

## 12. Seller Analysis (top 10 by revenue)

| Seller ID | Revenue (R$) | Orders | Avg Review |
|---|---:|---:|---:|
| 4869f7a5dfa277a7dca6462dcf3b52b2 | 249,393.44 | 1,131 | 4.12 |
| 7c67e1448b00f6e969d365cea6b010ab | 239,536.44 | 982 | 3.34 |
| 53243585a1d6dc2643021fd1853d8905 | 235,856.68 | 358 | 4.07 |
| 4a3ca9315b744ce9f8e9374361493884 | 235,359.30 | 1,804 | 3.78 |
| fa1c13f2614d7b5c4749cbc52fecda94 | 202,861.67 | 584 | 4.33 |
| da8622b14eb17ae2831f4ac5b9dab84a | 185,192.32 | 1,314 | 4.06 |
| 7e93a43ef30c4f03f38b393420bc753a | 178,838.42 | 332 | 4.23 |
| 1025f0e2d44d7041d6cf58b6550e0bfa | 172,860.69 | 915 | 3.83 |
| 7a67c85e85bb2ce8582c35f2203ad736 | 162,607.24 | 1,159 | 4.22 |
| 955fee9216a65b617aa5c0531780ce60 | 160,582.90 | 1,286 | 4.04 |

| Metric | Final Value |
|---|---:|
| Sellers analyzed (with valid-order revenue) | 3,053 of 3,095 |
| Sellers generating 80% of revenue | 557 (18%) |

Note: seller `7c67e1448b00f6e969d365cea6b010ab` is 2nd by revenue but has a
below-average review score (3.34) — flagged in Business Insights as an account
management priority.

## 13. Delivery / Logistics

| Metric | Final Value | Interpretation |
|---|---:|---|
| Delivered orders analyzed | 96,474 | Excludes the 2 implausible-delivery-time orders |
| Late rate (vs. estimated delivery date) | 6.8% | |
| Average delivery time | 12.1 days | |
| Median delivery time | 10 days | |
| Late orders (n) | 6,533 | |
| On-time orders (n) | 89,941 | |

## 14. Review Distribution

| Score | Share |
|---|---:|
| 1 star | 11.9% |
| 2 stars | 3.2% |
| 3 stars | 8.3% |
| 4 stars | 19.2% |
| 5 stars | 57.4% |
| Average | 4.07 / 5 |

## 15. Payment Analysis

| Payment Type | Revenue (R$) | Orders | Revenue Share |
|---|---:|---:|---:|
| credit_card | 12,350,903.3 | 74,116 | 78.5% |
| boleto | 2,826,802.3 | 19,539 | 18.0% |
| voucher | 349,063.7 | 3,037 | 2.2% |
| debit_card | 212,367.8 | 1,514 | 1.3% |

## 16. Association Rules (Basket Analysis)

| Metric | Final Value |
|---|---:|
| Orders with 0 detectable categories (missing translation) | 1,394 |
| Orders with 1 category | 96,080 (97.8%) |
| Orders with 2 categories | 710 |
| Orders with 3 categories | 15 |
| Multi-category orders (>1) | 725 |
| Transactions used for Apriori | 725 |
| Minimum support threshold | 0.01 (1%) |
| Frequent itemsets found | 48 |
| Minimum lift threshold | 1.0 |
| Rules found | 22 |

Top rules by lift:

| Antecedent | Consequent | Support | Confidence | Lift |
|---|---|---:|---:|---:|
| perfumery | health_beauty | 0.0152 | 0.440 | 4.62 |
| health_beauty | perfumery | 0.0152 | 0.159 | 4.62 |
| bed_bath_table | home_confort | 0.0593 | 0.217 | 3.15 |
| home_confort | bed_bath_table | 0.0593 | 0.860 | 3.15 |
| baby | toys | 0.0262 | 0.204 | 2.96 |
| toys | baby | 0.0262 | 0.380 | 2.96 |
| baby | cool_stuff | 0.0276 | 0.215 | 2.40 |
| cool_stuff | baby | 0.0276 | 0.308 | 2.40 |
| sports_leisure | health_beauty | 0.0193 | 0.209 | 2.20 |
| health_beauty | sports_leisure | 0.0193 | 0.203 | 2.20 |

Interpretation of lift: a lift of 4.62 means the pair co-occurs about 4.62x more often
than would be expected if the two categories were purchased independently. It is not a
probability statement about individual customers.

## 17. Forecasting

| Metric | Final Value |
|---|---:|
| Forecast target | Monthly revenue (valid orders) |
| Aggregation frequency | Calendar month |
| Total stable months available | 20 |
| Training months | 17 |
| Test months (held out) | 3 (2018-06, 2018-07, 2018-08) |
| Prophet configuration | `yearly_seasonality=True, weekly_seasonality=False, daily_seasonality=False` |
| Seasonal-naive method | Same calendar month one year earlier; falls back to last training value if unavailable |
| **Prophet MAE** | R$ 894,262.8 |
| **Prophet MAPE** | **88.1%** |
| **Seasonal-naive MAE** | R$ 438,837.6 |
| **Seasonal-naive MAPE** | **43.0%** |
| Better out-of-sample model | Seasonal-naive baseline |
| Forward forecast horizon | 6 months |
| Forward forecast average | R$ 1,492,523 |
| Historical average (20 stable months) | R$ 784,358 |

## 18. Statistical Tests

| Test | Method | Statistic | p-value | n | Result |
|---|---|---:|---:|---:|---|
| Delivery days vs. review score | Pearson correlation | r = -0.336 | < 0.001 (reported as 0.0e+00, floating-point underflow) | 96,474 | Significant negative association |
| Late vs. on-time review score | Welch's t-test (unequal variance) | t = -102.463 | < 0.001 | 96,474 (6,533 late / 89,941 on-time) | Significant difference; late orders average 2.26 vs. 4.28 |
| Payment installments vs. order value | Pearson correlation | r = 0.319 | < 0.001 | 99,441 | Significant positive association |

All three tests use the full or near-full order population, so the p-values are
expected to be extremely small given the sample size — statistical significance here
should not be read as effect size. The correlation coefficients (r = -0.336 and r =
0.319) are the more informative numbers for judging practical relevance.
