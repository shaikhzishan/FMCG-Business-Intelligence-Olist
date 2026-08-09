# Reproducibility

## 1. Dataset acquisition

This project uses the public **Brazilian E-Commerce Public Dataset by Olist**. Download
it from Kaggle (search "Olist Brazilian E-Commerce" or use the dataset owner's public
listing) or any legitimate public mirror of the same files.

Expected files (all CSVs):

```
olist_customers_dataset.csv
olist_orders_dataset.csv
olist_order_items_dataset.csv
olist_order_payments_dataset.csv
olist_order_reviews_dataset.csv
olist_products_dataset.csv
olist_sellers_dataset.csv
olist_geolocation_dataset.csv
product_category_name_translation.csv
```

Expected row counts, to confirm you have the same version of the dataset the notebook
was validated against:

| File | Rows |
|---|---:|
| `olist_orders_dataset.csv` | 99,441 |
| `olist_order_items_dataset.csv` | 112,650 |
| `olist_order_payments_dataset.csv` | 103,886 |
| `olist_order_reviews_dataset.csv` | 100,000 |
| `olist_products_dataset.csv` | 32,951 |
| `olist_sellers_dataset.csv` | 3,095 |
| `product_category_name_translation.csv` | 71 |

## 2. Directory structure

Place all CSVs in a single folder. The notebook expects this path by default (matching
the standard Kaggle mount point):

```
/kaggle/input/datasets/shaikhzishan982/fmcg-business-intelligence-dataset/
```

To run outside Kaggle, set an environment variable instead of moving files:

```bash
export OLIST_DATA_PATH=/path/to/your/csv/folder
```

The notebook reads `DATA_PATH = Path(os.environ.get("OLIST_DATA_PATH", <kaggle default>))`
at the top of the Data Preparation section, so this is the only path configuration
needed.

## 3. Python environment

Tested with Python 3.12. Required packages (no specific pinned versions were verified
beyond "the versions available via `pip install <package>` at the time of writing" —
do not assume specific version numbers unless you've confirmed them yourself):

```bash
pip install pandas numpy matplotlib seaborn plotly scikit-learn scipy statsmodels prophet mlxtend jupyter
```

Prophet depends on `cmdstanpy`, which downloads and compiles a small Stan binary on
first import — this is normal and only happens once per environment.

## 4. Random seed

`RANDOM_STATE = 42` is set once near the top of the notebook and reused everywhere
randomness is involved (K-Means initialization, PCA). This does not affect the Prophet
or seasonal-naive forecast, which are deterministic given the same input data.

## 5. Execution instructions

**Local / any Jupyter environment:**
1. Confirm the CSVs are in place per section 2.
2. Open the notebook.
3. Kernel → Restart & Run All.

**Command line (matches how this project was validated):**
```bash
OLIST_DATA_PATH=/path/to/csvs jupyter nbconvert --to notebook --execute notebook.ipynb --output executed.ipynb --ExecutePreprocessor.timeout=1800
```

**Kaggle:**
1. Create a new notebook and attach the Olist dataset.
2. Upload/paste the notebook content.
3. If the dataset attaches at a different path than the default, set
   `OLIST_DATA_PATH` in a cell before the Data Preparation section, or edit the
   default path directly.
4. Run All.

## 6. Expected runtime

On the environment this project was validated in (containerized, no GPU), a full
Restart-Kernel-→-Run-All execution against the real 99,441-order dataset took
approximately 12-15 minutes, dominated by:
- Cohort/RFM aggregation over ~95,000 customers
- K-Means silhouette search across K=2 through K=8
- Apriori itemset mining
- Two Prophet model fits (backtest + final refit)

Runtime will vary by hardware; none of these steps require a GPU.

## 7. Notes on determinism

Given the same input CSVs, the notebook was confirmed to produce **consistent results
across two consecutive full executions** after the final code fixes were applied
(see `VALIDATION.md`). Minor floating-point-level differences in library internals
(e.g. Prophet's Stan backend) are possible across different hardware/OS combinations
but should not materially change any reported figure.
