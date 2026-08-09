# Business Insights

Each insight below follows: **Finding** (the actual result) → **What it means** (plain
interpretation) → **Why it matters** → **Recommended action**. Numbers match
`ANALYTICAL_RESULTS.md` exactly.

---

## 1. A naive table join inflates revenue by 1.29x

**Finding:** Chaining `orders → order_items → payments` and summing `payment_value`
gives R$20,308,135. The corrected order-level total is R$15,739,137.

**What it means:** `order_items` and `payments` both have more than one row per order
(1.14 and 1.04 rows/order respectively). A direct merge duplicates `payment_value` once
per extra row.

**Why it matters:** Any KPI built on the naive total — revenue, AOV, CLV, growth rate —
would be wrong by roughly this factor, and the error is silent; nothing in a standard
`.merge()` call warns you about it.

**Recommended action:** Not a business action — a data-engineering practice: check rows-per-key
before joining any table with a payments/line-item structure, on this dataset or any
other with a similar schema.

---

## 2. Average order value is right-skewed

**Finding:** Mean order value R$160.26, median R$105.28.

**What it means:** A relatively small number of large orders pull the average up well
above the typical order.

**Why it matters:** Reporting the mean alone overstates what a "typical" customer
spends — relevant for anything from marketing budget assumptions to customer-value
segmentation cutoffs.

**Recommended action:** Use the median for "typical order" framing in business
communication; reserve the mean for total-revenue math where it's the mathematically
correct input.

---

## 3. A growth calculation bug produced a meaningless 33,463% figure

**Finding:** Growth computed against Olist's near-empty launch months (2 orders in
Sept 2016) gave an average monthly growth of ~33,463%. Restricting the denominator to
months at or above 10% of peak volume corrects this to 14.4%.

**What it means:** The underlying orders in those launch months are real — they're just
not a meaningful baseline for a growth-rate calculation. This is a modeling decision
about which months are informative, documented explicitly rather than silently applied.

**Why it matters:** A growth figure like 33,463% would be immediately spotted as wrong
by anyone reviewing it, which is exactly why it needed a real fix, not a rounding
change or an ad hoc exclusion of "the first row."

**Recommended action:** For any time series with a genuine launch/ramp period, define
"stable" by relative volume, not by literal position (first/last row) — this catches
sparse months anywhere in the series, not just at the very edges.

---

## 4. Repeat purchase is rare

**Finding:** Mean order frequency per customer is 1.034; cohort retention drops from
100% (month 0) to 5.2% (month 1) to under 1% from month 2 onward.

**What it means:** The overwhelming majority of Olist customers in this window bought
once and did not return within the observation period.

**Why it matters:** This shapes what kind of customer-value strategy makes sense — a
loyalty program assumes customers who repeat; this data shows most don't.

**Recommended action:** Prioritize converting a first purchase into a second one
(post-purchase incentive, timely follow-up) over a loyalty program built for repeat
buyers who barely exist in this dataset yet.

---

## 5. Late delivery is associated with meaningfully lower review scores

**Finding:** r = -0.336, p < 0.001 (n = 96,474); late orders average 2.26 stars vs. 4.28
for on-time orders (Welch's t-test, t = -102.463, p < 0.001).

**What it means:** There is a real, statistically robust association between delivery
delay and satisfaction in this data.

**Why it matters:** Of everything tested in this project, this is the relationship with
the clearest statistical backing — worth acting on with more confidence than the
descriptive-only findings.

**Recommended action:** This is a correlation, not a proven cause — orders that run
late could share other factors (seller, category) that independently affect reviews.
That said, given the strength of the association, prioritizing delivery-time
improvement in the slowest states (see #7) is a reasonable, evidence-backed bet, not a
leap.

---

## 6. Revenue and seller base are concentrated

**Finding:** Top 3 states = 63% of revenue; 557 of 3,053 sellers (18%) generate 80% of
revenue.

**What it means:** This is a fairly typical Pareto pattern for a marketplace and, on
its own, isn't surprising — the useful next question is what distinguishes the
concentrated group.

**Why it matters:** A platform-wide initiative is a blunt tool when the value is this
concentrated; targeted attention to a specific, identifiable group is more efficient.

**Recommended action:** Cross-reference the top-revenue sellers against review score
(seller `7c67e1448b00f6e969d365cea6b010ab`, for example, is 2nd by revenue but has a
3.34 average review — a below-average score on a high-revenue account) to flag
outsized reputational risk.

---

## 7. The K-Means split is driven by frequency and spend, not satisfaction

**Finding:** Silhouette-selected K=2. Cluster 0 (buy-once, R$160.52 avg spend, 4.10 avg
review) vs. Cluster 1 (repeat, R$330.07 avg spend, 4.14 avg review).

**What it means:** Average review score is nearly identical between the two clusters —
satisfaction isn't what separates high- and low-value customers here.

**Why it matters:** This rules out "unhappy customers are the low-value ones" as an
explanation for the repeat-purchase gap — the data doesn't support that story, so it
shouldn't be assumed in a strategy conversation.

**Recommended action:** Look elsewhere (category type, price point, first-order
experience) for what predicts repeat purchase, since satisfaction alone doesn't explain
the split in this data.

---

## 8. A small number of category pairs show real co-purchase patterns

**Finding:** 22 association rules from 725 multi-category orders; strongest is
perfumery ↔ health & beauty (lift 4.62, support 1.5%).

**What it means:** When a customer does buy across categories, certain pairs occur
meaningfully more often than chance — but this covers only 2.2% of otherwise-valid
orders.

**Why it matters:** These are real patterns, not noise (lift is well above 1 for all 22
rules), but the low support means they describe a minority behavior, not the typical
Olist purchase.

**Recommended action:** Treat these pairs as A/B-testable cross-sell hypotheses (e.g.
"customers viewing perfumery items might respond to a health & beauty recommendation"),
not as an established, high-confidence merchandising rule.

---

## 9. A validated forecast comparison found the simple model wins

**Finding:** Seasonal-naive baseline MAPE 43.0% vs. Prophet MAPE 88.1%, backtested on 3
held-out months.

**What it means:** With only 17 months of training data, Prophet doesn't have enough
history to learn a reliable yearly seasonal pattern — it appears to extrapolate the
sharp 2017 growth trend into a test period that had actually leveled off.

**Why it matters:** This is exactly the kind of result that's tempting to hide or spin
as "the forecast shows continued growth" — but the honest reading is that the more
sophisticated model underperformed a naive rule here, and reporting the number that
way is more useful to a reader than reporting the fancier-sounding one.

**Recommended action:** Use the seasonal-naive baseline for near-term revenue planning;
re-run this comparison every few months as training history grows, since Prophet may
become the better choice once it has more seasonal cycles to learn from.
