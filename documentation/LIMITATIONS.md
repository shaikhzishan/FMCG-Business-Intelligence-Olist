# Limitations

## Data limitations

- **No cost or margin data.** Every revenue figure in this project (R$15,739,137
  total, R$160.26 AOV, category and seller revenue tables) is payment value collected,
  not profit or margin. Olist's public dataset doesn't include cost-of-goods, so no
  profitability calculation was attempted anywhere in this project, and none should be
  inferred from a "revenue" figure.
- **Anonymized customer identifiers.** There's no acquisition channel, marketing spend,
  or campaign data linked to any customer. Customer analysis is limited to what can be
  derived from order behavior alone (recency, frequency, monetary value, review score).
- **`customer_id` vs. `customer_unique_id`.** The dataset assigns a new `customer_id`
  per order, even for repeat customers — `customer_unique_id` is the real person-level
  key, and all repeat-purchase analysis (RFM, cohorts, CLV) uses it. This is documented
  because it's an easy mistake to make with this dataset if not caught early.
- **Limited historical window.** Data spans September 2016 to October 2018 (about two
  years), with the first several months representing platform launch/ramp-up rather
  than steady-state operation. This constrains how much can be said about
  long-run seasonality or trend.

## Analytical limitations

- **Observational associations, not causal claims.** The delivery-delay/review-score
  relationship (r = -0.336, p < 0.001) and the installments/order-value relationship
  (r = 0.319, p < 0.001) are both statistically significant associations. Neither is
  backed by an experimental design (e.g. random assignment of delivery speed), so
  neither is presented as a causal effect anywhere in this project.
- **Review score is order-level, shared across items.** For multi-item orders, the same
  review score is attached to every item when computing category-level review
  averages. This is a known approximation, disclosed directly rather than hidden.
- **Association rules have low support.** All 22 rules found have support under 6% —
  they describe real but minority co-purchase patterns (2.2% of otherwise-valid orders
  span more than one category at all). They should be treated as cross-sell
  hypotheses to test, not established purchasing behavior.
- **K-Means found only 2 clusters.** Given how much of the customer base is compressed
  into "bought once, average spend" (97% of customers), the silhouette-optimal
  clustering doesn't produce a rich multi-segment structure. This is a genuine finding
  about this data, not a modeling shortfall — more clusters were tested (K=2 through
  K=8) and none scored higher.

## Forecast limitations

- **High out-of-sample error.** Both models tested had substantial error on the 3-month
  holdout: seasonal-naive 43.0% MAPE, Prophet 88.1% MAPE. Neither should be used for a
  precise revenue commitment — the notebook's own conclusion is to prefer the simpler
  model, and even that carries meaningful uncertainty.
- **Insufficient history for reliable seasonality.** 17 training months is under one
  and a half years — not enough for Prophet (or likely any seasonal model) to learn a
  dependable yearly pattern. This is stated as the likely cause of Prophet's weaker
  performance, not glossed over.
- **Forecast reflects platform-stage dynamics, not steady-state demand.** Because the
  training window includes Olist's rapid 2017 scaling phase, any model trained on it
  risks extrapolating that growth curve rather than a stabilized demand pattern.

## Scope limitations

- **This is a public marketplace dataset, not internal company data.** Olist itself is
  a marketplace connecting sellers to retail channels, not an FMCG company — this
  project treats the underlying business questions (revenue, customer value, category
  and seller performance, delivery, forecasting) as representative of retail/e-commerce
  BI work generally, without claiming Olist is an FMCG business.
- **No external benchmark data.** There's no reliable, citable industry benchmark (AOV,
  retention rate, delivery SLA) available to compare these findings against, so all
  comparisons in this project are internal to the dataset itself.
- **Individual portfolio project.** This is not a consulting engagement and doesn't
  represent measured business outcomes, implemented recommendations, or stakeholder
  requirements — none of that is claimed anywhere in the documentation.
