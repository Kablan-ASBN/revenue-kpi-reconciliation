# Revenue & KPI Reconciliation
### Order-Level Revenue Definitions, Coverage, and Validation

![Python](https://img.shields.io/badge/Python-3.10+-blue) ![Pandas](https://img.shields.io/badge/Pandas-2.0-blue) ![Status](https://img.shields.io/badge/Status-Complete-brightgreen) ![Context](https://img.shields.io/badge/Context-Analytics%20Engineering%20Portfolio-informational)

---

## The Problem

Finance, Operations, and Marketing report different revenue numbers from the same source data. This is one of the most common and costly data trust failures in analytics. The instinct is to assume bad data. The reality is usually something more subtle: different teams are answering different questions with the same word — "revenue" — and nobody documented the difference.

This project investigates that problem directly on the Olist Brazilian E-Commerce dataset. The goal is not to find the "correct" revenue number. It is to define each metric precisely, quantify the gaps between them, classify every order-level discrepancy by root cause, and build validation checks that can run in production to catch issues before they reach stakeholders.

---

## Dataset

**Source:** [Olist Brazilian E-Commerce Dataset](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce) — publicly available on Kaggle.

Four core tables used in this analysis:

| Table | Grain | Purpose |
|---|---|---|
| `olist_orders_dataset` | One row per order | Order status and timestamps |
| `olist_order_items_dataset` | One row per item | Item prices and freight charges |
| `olist_order_payments_dataset` | One row per payment installment | Captured payment values |
| `olist_customers_dataset` | One row per customer | Customer identity deduplication |

**Key grain risk:** The items and payments tables are multi-row per order. Joining them directly without first aggregating to order level produces fan-out and inflated revenue totals. This is identified and resolved in the pipeline before any revenue calculation.

---

## Pipeline Overview

### 1. Data Ingestion and Verification
Load raw tables and run schema sanity checks confirming all required columns are present before any transformation begins. Raw data is treated as immutable.

### 2. Grain and Duplication Analysis
Confirm the grain of each table before joining. Items average multiple rows per order (multi-item orders). Payments average multiple rows per order (installments and split payments). Both tables are aggregated to order level before the reconciliation join, preventing the most common source of inflated revenue metrics.

### 3. Order-Level Join
Merge orders, aggregated items, and aggregated payments using left joins with `validate="one_to_one"` to assert grain integrity at every step. The result is one row per order with all revenue dimensions on a single line.

### 4. Revenue Definitions
Four distinct revenue metrics are defined, computed, and compared:

| Definition | Formula | Typical Owner |
|---|---|---|
| Gross revenue (items only) | Sum of item prices per order | Operations |
| Gross revenue (items + freight) | Item prices + freight charges | Finance |
| Paid revenue (all) | Sum of captured payment values | Finance / Payments |
| Net revenue proxy | Paid revenue excluding canceled orders | Executive reporting |

Refund data is not available in this dataset. Canceled orders with captured payments are used as a conservative proxy for net revenue adjustment.

### 5. Mismatch Bucketing
Every order is classified into one of five mutually exclusive mismatch buckets based on the relationship between item + freight revenue and paid revenue:

| Bucket | Definition | Severity |
|---|---|---|
| `match_within_tolerance` | Delta within $0.05 USD (rounding / split payment residuals) | None |
| `payment_no_items` | Payment captured but no items found | High |
| `canceled_with_payment` | Order canceled but payment was captured | Medium |
| `items_freight_payment_delta_remaining` | Items and payment both exist, non-trivial delta remains | Low |
| `items_no_payment` | Items exist but no payment recorded | High |

### 6. Validation Checks
Seven named data quality checks run against the reconciled dataset, each with a defined severity level:

| Check | Severity |
|---|---|
| Orders without items | High |
| Orders without payments | High |
| Payments referencing unknown order IDs | Critical |
| Items referencing unknown order IDs | Critical |
| Canceled orders with positive payment | Medium |
| Multi-payment orders | Low |
| Multi-item orders | Low |

---

## Results

### Revenue by Definition (Order-Level)

![Revenue by Definition](revenue_by_definition__order-level_.png)

The $2.4M gap between gross items-only revenue (~$13.6M) and paid revenue (~$16.0M) is entirely explained by freight charges and definition scope — not data errors. Once freight is included in the gross figure, all four definitions converge within $0.2M.

### Mismatch Buckets — Order Share

![Mismatch Buckets Order Share](Mismatch_Buckets__order_share_.png)

Over 99% of orders fall within the match tolerance. The remaining mismatch volume is concentrated in two buckets: `payment_no_items` (orders where payment was captured but no item record exists) and `canceled_with_payment` (orders canceled after payment was collected).

### Mismatch Buckets — Absolute Revenue Delta

![Mismatch Buckets Revenue Delta](Mismatch_Buckets__Absolute_revenue_delta_.png)

The `payment_no_items` bucket drives the largest absolute revenue impact (~$0.13M), despite representing less than 1% of orders. This is the highest-priority operational issue identified — a small number of orders account for a disproportionate share of unexplained revenue.

---

## Key Findings

**Finding 1: Most revenue discrepancies are definition differences, not data problems.** The $2.4M gap between gross items-only and paid revenue is fully explained by freight charges and canceled order treatment. Teams disagreeing on revenue are often measuring different things correctly rather than measuring the same thing incorrectly.

**Finding 2: A small number of orders drive most unexplained discrepancy.** The `payment_no_items` bucket — less than 1% of orders — accounts for ~$130K in unexplained revenue delta. In a production setting this would be the first escalation to the payments and order management teams.

**Finding 3: Canceled orders with captured payments are a systematic reporting gap.** These orders appear in paid revenue totals but represent transactions that did not complete. Without explicit refund data, they inflate reported revenue in systems that do not filter by order status.

**Finding 4: Multi-item and multi-payment joins are the primary technical risk.** Joining items and payments directly to orders without prior aggregation produces fan-out and inflated totals. This is the most common grain error in revenue reporting and is explicitly prevented in this pipeline.

---

## Known Limitations

- **No refund data.** The dataset does not include explicit refund records. Canceled orders with payments are used as a proxy, which understates true net revenue if some canceled orders were later refunded.
- **Freight not always separately reported.** Some operational systems embed freight in item price rather than separating it, which would shift the baseline for the items-only definition.
- **Static snapshot.** This is a point-in-time analysis. A production implementation would run incrementally against new orders each reporting cycle.

---

## Project Structure

```
revenue-kpi-reconciliation/
├── README.md
├── revenue_reconciliation_order_level.ipynb      # Full analysis notebook
├── revenue_by_definition__order-level_.png       # Revenue comparison chart
├── Mismatch_Buckets__order_share_.png            # Mismatch distribution chart
├── Mismatch_Buckets__Absolute_revenue_delta_.png # Revenue delta chart
└── data/
    └── processed/
        ├── reconciliation_order_level.csv        # Order-level reconciliation output
        └── validation_checks.csv                # Validation check results
```

---

## Reproducing This Project

The raw Olist dataset is not included due to file size. To reproduce:

1. Download the dataset from [Kaggle](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce)
2. Place CSV files in `data/raw/olist_dataset/olist/`
3. Run `revenue_reconciliation_order_level.ipynb` end to end

All processed outputs and visualizations are included in the repository.

---

## Takeaway

Revenue reconciliation is not a data cleaning problem — it is a metric definition problem. The same transaction data produces materially different totals depending on whether freight is included, how cancellations are treated, and at what grain the aggregation happens. This project builds the framework to make those differences explicit, measurable, and monitorable in production.

Part of a portfolio focused on analytics engineering, data quality, and production-grade reporting systems.

**Author:** Kablan Assebian · [GitHub](https://github.com/Kablan-ASBN) · [LinkedIn](https://www.linkedin.com/in/gomis-kablan/)
