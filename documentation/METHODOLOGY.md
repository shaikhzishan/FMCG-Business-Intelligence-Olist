# Methodology

This document explains why each method was chosen, not just what code ran. See
`ANALYTICAL_RESULTS.md` for the numeric output of each method.

## Revenue calculation

Revenue is defined as `payment_value` summed to the order level, for orders not in
`canceled` or `unavailable` status. It is **not** GMV in the sense of listed item price
— it's what was actually paid, including freight and any installment-related
adjustments. It is **not** profit — Olist doesn't publish cost data, so there's no
margin figure to compute, and none is implied anywhere in this project.

The reason revenue has to be computed from a pre-aggregated `order_payments` table
rather than a direct merge is structural: `order_payments` averages 1.04 rows per
order (installment splits), so summing `payment_value` on a table that's already been
joined to `order_items` (1.14 rows per order) multiplies the payment value once per
item row. Aggregating payments to the order grain *before* any join removes this risk
by construction, and an `assert` on `order_facts` row count catches it if it ever
recurs.

## Customer metrics (RFM base table)

Recency, frequency, and monetary value are computed per `customer_unique_id` (not
`customer_id`, which is order-specific in this dataset) from valid orders only.
Recency uses a snapshot date one day after the last valid order in the dataset, which
is standard practice so that the most recent buyer has recency = 1, not 0.

## RFM segmentation

Recency and frequency are each split into quintiles (1-5), then mapped to a named
segment using a standard 25-cell (R × F) grid rather than a set of hand-picked
threshold rules. The reason for the grid over ad-hoc rules: hand-picked thresholds
tend to leave a large "everything else" bucket that doesn't map to any decision, and
they're harder to defend than a documented, widely-used scheme. Monetary value (M) is
scored but not used in the segment label itself — the R×F grid is the standard
formulation and adding an M axis would require a 125-cell grid with far less
established interpretation.

## Customer lifetime value

CLV here is historical/realized value — total valid revenue attributed to a customer
to date — not a predictive/forward-looking estimate. A predictive CLV model would need
acquisition-time features (channel, first-order category, etc.) that don't exist in
this anonymized dataset, so it wasn't attempted. Median is reported as the headline
figure over mean because the distribution is right-skewed (max observed value is
~82x the mean), and the mean is disproportionately influenced by a small number of
high-value customers.

## Cohort retention

Cohort month is each customer's first valid-order month; cohort index is the number of
months between a given order and that cohort month. The table is built on deduplicated
order-level records specifically so that a customer with a multi-item order isn't
double-counted as having purchased twice in the same month — this was a real risk given
the row-multiplication issue documented in the join-grain section.

The averaged retention curve only includes cohort-index columns with at least 3
supporting cohorts. Early or late cohort-index values can be driven by a single small
cohort, which would otherwise dominate the average with a noisy, non-representative
number.

## K-Means segmentation

Recency, frequency, monetary value, and average review score are standardized
(z-scored) before clustering, since they're on very different scales (days vs. order
counts vs. R$ vs. a 1-5 score) and K-Means is distance-based. K is selected by
silhouette score over a candidate range of 2-8 rather than assumed — this is a
deliberate departure from picking a "nice" number of segments, and in this dataset it
selected K=2, which is a legitimate result given how much of the customer base is
compressed into "bought once" (97% of customers).

## Product/category and seller analysis

These use `item_level`, not `order_facts`, because "revenue" at the product or seller
grain means the item's own price (plus freight), not the order's total payment — a
seller only earns their own item's price, not whatever else was in the same order.
Review score is attached at the order level and therefore shared across all items in a
multi-item order; this is disclosed directly in the notebook as a known approximation,
not hidden.

## Delivery and statistical testing

Delivery time and lateness are computed at the order level (`order_facts`), since
delivery is a property of the whole order, not an individual item. Two orders with
implausible delivery times (negative or over 200 days) are excluded from delivery-time
analysis specifically — they're documented rather than silently dropped from the
broader dataset.

The relationship between delivery delay and review score is tested two ways: a Pearson
correlation (continuous delay vs. continuous score) and a Welch's t-test (late vs.
on-time, unequal-variance version since the two groups have very different sample
sizes — 6,533 vs. 89,941). Both are reported with exact statistics rather than just "a
significant relationship exists," and both are explicitly described as an association,
not a causal claim — there's no experimental design here (e.g. random assignment of
delivery speed) that would license a causal interpretation.

## Association rule mining

Apriori requires enough multi-item "baskets" to find patterns; single-category orders
contribute nothing to a category-level basket analysis, so the transaction set is
restricted to the 725 orders spanning more than one category (2.2% of otherwise-valid
orders touching multiple categories). Support threshold of 1% and lift threshold of
1.0 are standard starting points for a first pass — support below 1% risks fitting
noise from a handful of orders; a lift threshold of 1.0 keeps only pairs that co-occur
more than chance would predict, discarding negative or neutral associations. Given the
small transaction count, every rule found still has low absolute support (under 6%),
which is disclosed directly rather than glossed over — these are cross-sell hypotheses,
not established purchasing behavior.

## Seasonal decomposition

A classical additive decomposition (trend/seasonal/residual) requires at least two full
seasonal cycles to fit a 12-month seasonal component reliably — that's a minimum of 24
data points, and this dataset has only 20 stable months after excluding partial
launch/tail months. The notebook checks this explicitly and skips decomposition with an
explanation rather than fitting an unreliable model and presenting its output as if it
were trustworthy.

## Forecasting and baseline comparison

The forecast target is monthly revenue from valid orders. Before fitting a
forward-looking model, the last 3 of the 20 stable months are held out as a test set,
and two models are fit on the remaining 17 training months: Prophet (with yearly
seasonality enabled, weekly/daily disabled since the data is monthly) and a
seasonal-naive baseline (same calendar month one year earlier, falling back to the most
recent training value where a year-ago observation isn't available).

The reason for including the naive baseline isn't procedural — it's the actual test of
whether Prophet is worth using at all. A model that can't beat "just repeat what
happened last year" isn't earning its complexity. In this case, the naive baseline won
(43.0% MAPE vs. Prophet's 88.1%), which is reported as the primary forecasting
conclusion, not hidden or minimized. The reason Prophet likely struggled: 17 monthly
data points is under one and a half years, which isn't enough for it to estimate a
yearly seasonal component with any reliability — it appears to extrapolate the sharp
2017 growth trend forward instead, overshooting the more stable 2018 pattern the test
months actually show.

Given this result, the notebook's recommendation is to use the seasonal-naive baseline
for near-term planning and re-run this comparison periodically as more training months
accumulate, rather than defaulting to the more sophisticated model on the assumption
that it must be better.
