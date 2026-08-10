# FMCG Business Intelligence & Customer Analytics — Olist E-Commerce

Business intelligence analysis of the Olist Brazilian e-commerce dataset: **99,441 orders** placed between September 2016 and October 2018, across **96,096 unique customers, 3,095 sellers, and 71 product categories**.

I'm treating this as an FMCG/e-commerce BI case study. The questions addressed here — revenue trends, customer value, retention, category and seller performance, delivery reliability, and forecast accuracy — are the same types of questions a BI analyst would answer for a retail or FMCG business. Olist itself is a marketplace, not an FMCG company; the analytical questions are what make this a representative case study.

---

## Business Questions

* How has revenue moved over the observation window, and is the movement seasonal or structural?
* Which customers generate most of the value, and do they come back?
* Which product categories and sellers actually drive revenue?
* Is delivery performance related to how customers rate their orders?
* Can near-term revenue be forecast with any reliability from approximately two years of data?

---

## Dataset

Nine relational CSVs from the public Olist dataset:

* Customers
* Orders
* Order items
* Products
* Sellers
* Payments
* Reviews
* Geolocation
* Product category translation

The final analysis contains **99,441 orders, 112,650 order items, 103,886 payment records, and 100,000 review records**.

The raw dataset is **not stored in this repository** due to size. See the reproducibility documentation for instructions on obtaining and using the data.

---

## Methodology

A major focus of the project was maintaining the correct analytical grain.

`order_items` and `payments` both contain multiple rows per order on average — approximately **1.14 order-item rows** and **1.04 payment rows per order**.

A naive chained merge of:

```text
orders → order_items → payments
```

duplicates records and inflates revenue.

This was tested directly. The naive approach produced approximately **R$20.31M** in revenue compared with the correct order-level total of approximately **R$15.74M**, resulting in roughly **1.29× revenue inflation**.

To avoid this, the analysis builds separate correctly-grained tables:

* `order_facts` — one row per order for revenue, customer, and forecasting analysis
* `item_level` — one row per order item for product and seller analysis

Assertions and validation checks are used throughout the notebook to guard against accidental row multiplication.

---

## Key Findings

### Revenue

**R$15,739,137** in revenue/payment value across **98,207 valid non-cancelled orders**.

Average order value was **R$160.26**, compared with a median of **R$105.28**, showing a right-skewed order-value distribution.

### Growth

An initial growth calculation produced a meaningless **33,463%** average because it compared stable months against near-empty launch months.

After restricting the calculation to months with at least 10% of peak order volume, the corrected average monthly growth was **14.4%**.

This correction is important because the early Olist observation period contains extremely low-volume months that distort percentage-based growth calculations.

### Customer Retention

Approximately **97% of customers placed only one order**.

Cohort retention fell from 100% in the first month to approximately **5.2% by month 1**, and below **1% from month 2 onward**.

The data therefore shows a predominantly one-time customer base rather than a strong repeat-purchase pattern.

### Delivery & Customer Satisfaction

Delivery delay is significantly associated with lower review scores:

* **Correlation:** r = -0.336
* **p-value:** < 0.001
* **Observations:** n = 96,474
* Late-order average review: **2.26 stars**
* On-time average review: **4.28 stars**

This is an observational relationship and should not be interpreted as proof that delivery delay directly causes lower ratings.

### Revenue Concentration

Revenue is concentrated among a relatively small number of geographic markets and sellers:

* The top 3 states account for approximately **63% of revenue**.
* **557 of 3,053 sellers (18%)** account for approximately **80% of revenue**.

This concentration has implications for seller dependency and account-management priorities.

### Association Rules

Association-rule mining was performed on **725 multi-category orders**.

The analysis produced **22 rules**.

The strongest observed relationships included:

| Category relationship       |     Lift |
| --------------------------- | -------: |
| Perfumery ↔ Health & Beauty | **4.62** |
| Bed & Bath ↔ Home Comfort   | **3.15** |
| Baby ↔ Toys                 | **2.96** |

These relationships have relatively low support and should therefore be treated as cross-selling hypotheses rather than established purchasing behavior.

### Revenue Forecasting

A validated forecast comparison showed that the simpler **seasonal-naive baseline outperformed Prophet**:

| Model                   |      MAPE |
| ----------------------- | --------: |
| Seasonal-naive baseline | **43.0%** |
| Prophet                 | **88.1%** |

With only approximately **17 training months**, there is insufficient history for Prophet to reliably learn yearly seasonality.

For this dataset, the simpler baseline is therefore the more defensible choice for near-term directional planning.

---

## Business Recommendations

### 1. Improve Delivery Performance

Prioritize delivery-time improvements in slower states with meaningful order volume.

The delivery/review relationship is the clearest statistically supported relationship identified in the analysis, although it remains observational rather than causal.

### 2. Focus on First-to-Second Order Conversion

With approximately 97% of customers purchasing only once, a first-to-second-order incentive is likely a more relevant retention lever than building a loyalty program around an already-small repeat customer base.

### 3. Monitor High-Value Sellers

Cross-reference high-revenue sellers with review performance to identify accounts that generate significant revenue but underperform on customer satisfaction.

### 4. Test Cross-Selling Opportunities

The association-rule results provide potential product-page cross-selling opportunities.

The relationships should be tested experimentally because the observed support is low.

### 5. Keep Forecasting Simple for Now

The seasonal-naive model outperformed Prophet in backtesting.

Until more historical data becomes available, the simpler model should be preferred for near-term directional planning rather than adding complexity without demonstrated improvement.

---

## Limitations

* No cost or margin data is available, so revenue figures represent payment value rather than profit.
* Customer identifiers are anonymized.
* No acquisition-channel or marketing data is available.
* Delivery/satisfaction and category/satisfaction relationships are observational associations rather than causal estimates.
* Forecast error remains high, with MAPE ranging from approximately 43% to 88%.
* The relatively short historical period limits the ability of complex forecasting models to learn yearly seasonality.
* Association-rule support is below 6% for the reported rules, so the relationships should not be overinterpreted.
* Olist is an e-commerce marketplace dataset rather than an FMCG company's internal dataset; the FMCG framing is based on the business questions and analytical use cases.

---

## Technology Stack

**Python**

* pandas
* NumPy
* Matplotlib
* Seaborn
* Plotly
* scikit-learn
* SciPy
* Statsmodels
* Prophet
* mlxtend

---

## Repository Structure

```text
FMCG-Business-Intelligence-Olist/
│
├── README.md
├── requirements.txt
│
├── notebooks/
│   └── FMCG_BI_Olist_Analytics.ipynb
│
└── documentation/
    ├── PROJECT_DOCUMENTATION.md
    ├── ANALYTICAL_RESULTS.md
    ├── DATA_DICTIONARY.md
    ├── METHODOLOGY.md
    ├── VALIDATION.md
    ├── BUSINESS_INSIGHTS.md
    ├── LIMITATIONS.md
    └── REPRODUCIBILITY.md
```

---

## How to Reproduce

1. Obtain the Olist dataset CSV files.
2. Set `OLIST_DATA_PATH` to the directory containing the dataset files, or use the Kaggle-style path expected by the notebook.
3. Install the required Python packages:

```bash
pip install -r requirements.txt
```

4. Open:

```text
notebooks/FMCG_BI_Olist_Analytics.ipynb
```

5. Restart the kernel and run the notebook from beginning to end.

For detailed environment, dataset, and execution instructions, see:

`documentation/REPRODUCIBILITY.md`

---

## Documentation

Detailed project documentation is available in the `documentation/` directory:

* **Project Documentation** — complete technical overview
* **Analytical Results** — detailed numerical results
* **Data Dictionary** — source tables, fields, and analytical grain
* **Methodology** — analytical methods and reasoning
* **Validation** — data-quality and analytical validation
* **Business Insights** — findings translated into business actions
* **Limitations** — analytical and data limitations
* **Reproducibility** — instructions for reproducing the analysis

---

## Links

* **Kaggle Dataset:(https://www.kaggle.com/datasets/shaikhzishan982/fmcg-business-intelligence-dataset)** 
* **Kaggle Notebook:** 

---

## Project Status

**Completed — final notebook executed on the real Olist dataset.**

The analysis includes data-quality validation, customer analytics, revenue analysis, category and seller analysis, delivery/review analysis, association-rule mining, and validated revenue forecasting.

All reported figures in this repository are based on the final real-data execution of the notebook.
