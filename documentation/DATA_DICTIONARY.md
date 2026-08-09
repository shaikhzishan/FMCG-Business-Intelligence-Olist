# Data Dictionary

## Raw tables (Olist public dataset)

### `orders`
- **Grain:** one row per order (99,441 rows, confirmed 1.00 rows/order in the grain check).
- **Purpose:** the spine of the dataset — every other table joins back to this one via `order_id`.
- **Key columns:** `order_id` (PK), `customer_id` (FK → `customers`), `order_status`
  (delivered / shipped / canceled / processing / invoiced / unavailable / etc.),
  `order_purchase_timestamp`, `order_approved_at`, `order_delivered_carrier_date`,
  `order_delivered_customer_date`, `order_estimated_delivery_date`.
- **Notes:** delivery-related timestamps are null for orders that never shipped/delivered
  (3.0% null on `order_delivered_customer_date`) — expected, not a data defect.

### `order_items`
- **Grain:** one row per line item within an order (112,650 rows, 1.14 rows/order on average).
- **Purpose:** product/seller detail and pricing at the item level.
- **Key columns:** `order_id` (FK → `orders`), `order_item_id` (sequence number within the
  order), `product_id` (FK → `products`), `seller_id` (FK → `sellers`),
  `shipping_limit_date`, `price`, `freight_value`.
- **Analytical use:** product and seller revenue/performance. **Never** used to sum
  order-level revenue directly — see `item_level` below.

### `order_payments`
- **Grain:** one row per payment transaction; can be more than one row per order
  (installments), 1.04 rows/order on average.
- **Purpose:** payment method and amount.
- **Key columns:** `order_id` (FK → `orders`), `payment_sequential`, `payment_type`
  (credit_card / boleto / voucher / debit_card), `payment_installments`, `payment_value`.
- **Analytical use:** must be aggregated (summed) to the order grain before joining to
  `orders` for any revenue calculation — see `order_facts` below.

### `order_reviews`
- **Grain:** nominally one row per order, but the raw file (100,000 rows) has slightly
  more rows than unique orders (1.01 rows/order) — a small number of orders have more
  than one review record.
- **Purpose:** post-purchase satisfaction score and free-text comments.
- **Key columns:** `review_id`, `order_id` (FK → `orders`), `review_score` (1-5),
  `review_comment_title`, `review_comment_message`, `review_creation_date`,
  `review_answer_timestamp`.
- **Analytical use:** deduplicated to one review per order (most recent by
  `review_answer_timestamp`) before joining — see `order_facts` below. Free-text fields
  are not used in this project.

### `customers`
- **Grain:** one row per `customer_id`. Note `customer_id` is order-specific in this
  dataset (a repeat customer gets a new `customer_id` per order); `customer_unique_id`
  is the actual person-level identifier.
- **Purpose:** customer location.
- **Key columns:** `customer_id` (PK, FK from `orders`), `customer_unique_id` (the real
  customer key for repeat-purchase analysis), `customer_zip_code_prefix`,
  `customer_city`, `customer_state`.

### `products`
- **Grain:** one row per `product_id` (32,951 rows).
- **Purpose:** product category and physical attributes.
- **Key columns:** `product_id` (PK), `product_category_name` (Portuguese),
  `product_weight_g`, `product_length_cm`, `product_height_cm`, `product_width_cm`.

### `sellers`
- **Grain:** one row per `seller_id` (3,095 rows).
- **Purpose:** seller location.
- **Key columns:** `seller_id` (PK), `seller_zip_code_prefix`, `seller_city`, `seller_state`.

### `geolocation`
- **Grain:** many rows per zip-code prefix (not deduplicated in the source).
- **Purpose:** zip-code-to-lat/lng lookup.
- **Analytical use:** loaded but not used in the final analysis — state-level
  aggregation from `customers`/`sellers` was sufficient for the questions asked.

### `product_category_name_translation`
- **Grain:** one row per category (71 rows).
- **Purpose:** Portuguese → English category name mapping.
- **Key columns:** `product_category_name` (FK → `products`), `product_category_name_english`.

## Derived analytical tables (built in the notebook)

### `order_facts`
- **Grain:** one row per order — 99,441 rows, asserted equal to `orders.order_id.nunique()`.
- **Built from:** `orders` left-joined to `customers` (on `customer_id`), to
  `order_payments` pre-aggregated to order grain (summed `payment_value`, max
  `payment_installments`, dominant `payment_type`), and to `order_reviews`
  deduplicated to one row per order.
- **Key derived columns:** `revenue` (payment value, zeroed for canceled/unavailable
  orders), `is_valid` (boolean, excludes canceled/unavailable), `delivery_days`,
  `delivery_delay_days`, `is_late`, `delivery_flag` (implausible delivery time).
- **Used for:** all revenue, customer (RFM/CLV/cohort/K-Means), delivery, payment, and
  forecasting analysis.

### `item_level`
- **Grain:** one row per order item — 112,650 rows, asserted equal to `order_items` row count.
- **Built from:** `order_items` left-joined to a subset of `orders` columns, `customers`,
  `products`, `sellers`, and the category translation table.
- **Key derived columns:** `item_revenue` (price + freight_value), `is_valid`.
- **Used for:** product/category and seller analysis, and the basket/association-rule
  analysis. Never used to compute order-level revenue totals (that would reintroduce
  the row-multiplication risk this table structure exists to avoid).

### `monthly`
- **Grain:** one row per calendar month (24 rows total).
- **Built from:** `order_facts` (valid orders only), grouped by `purchase_year_month`.
- **Key derived columns:** `revenue`, `orders`, `aov`, `partial` (flag for months under
  10% of peak monthly order volume).

### `rfm`
- **Grain:** one row per `customer_unique_id` with at least one valid order (94,990 rows).
- **Built from:** `order_facts` (valid orders), grouped by customer.
- **Key derived columns:** `recency`, `frequency`, `monetary`, `avg_review`, `R`/`F`/`M`
  quintile scores, `segment` (RFM segment label).

### `cohort_pivot` / `retention`
- **Grain:** one row per acquisition cohort month (23 rows), one column per
  months-since-first-order.
- **Built from:** deduplicated order-level records from `order_facts`, grouped by each
  customer's first purchase month.

### `basket_df`
- **Grain:** one row per multi-category order (725 rows), one boolean column per
  product category present in the basket universe.
- **Built from:** `item_level` (valid orders, multi-category orders only, categories
  with a non-null translation), transaction-encoded via `mlxtend.TransactionEncoder`.
