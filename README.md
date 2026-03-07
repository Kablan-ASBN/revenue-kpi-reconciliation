# Revenue & KPI Reconciliation
### Order-Level Commerce Analytics Case Study

![Python](https://img.shields.io/badge/Python-3.10+-3776AB?style=flat&logo=python&logoColor=white)
![Pandas](https://img.shields.io/badge/Pandas-Data%20Wrangling-150458?style=flat&logo=pandas)
![Jupyter](https://img.shields.io/badge/Jupyter-Notebook-F37626?style=flat&logo=jupyter)
![Status](https://img.shields.io/badge/Status-Complete-2ea44f?style=flat)

## The Problem

Finance, Operations, and Marketing teams routinely report different revenue numbers from the same transaction data. This creates distrust in dashboards, wasted time in stakeholder meetings, and inconsistent decision-making.

The root cause is almost never bad data. It is implicit assumptions about definitions, aggregation grain, and which orders to include. This project makes those assumptions explicit, reconciles four revenue definitions against each other, and produces the validation checks needed to catch discrepancies automatically in production.

## Why This Matters for Analytics Engineering

Revenue reconciliation is a foundational AE problem. It requires:

- **Grain standardization:** aggregating item-, freight-, and payment-level data to a consistent order-level grain before any joins, to prevent row duplication and inflated metrics
- **Explicit metric definitions:** documenting the exact calculation logic for each revenue variant so that differences between teams become traceable, not mysterious
- **Validation logic:** designing checks that can be automated in a production pipeline to catch data quality issues before they reach stakeholders
- **Root cause thinking:** distinguishing between discrepancies caused by definition choice versus actual data errors

This is exactly the work that prevents "why don't our numbers match?" conversations from happening in the first place.

## Dataset

**Source:** [Brazilian E-Commerce Public Dataset (Olist)](https://www.kaggle.com/datasets/olistbr/brazilian-ecommerce), covering 100K+ real orders from 2016 to 2018.

| Table | Description |
|---|---|
| `orders` | Order lifecycle, timestamps, and status |
| `order_items` | Item-level prices and freight charges |
| `order_payments` | Customer payment records (supports installments and split payments) |
| `customers` | Customer identifiers (used for integrity checks) |

Raw data is treated as immutable. All transformations happen downstream.

## Revenue Definitions Compared

All metrics are calculated at the **order level** to prevent duplication from multi-item and multi-payment orders.

| Definition | What It Includes | Typical Use Case |
|---|---|---|
| **Gross (Items Only)** | Sum of item prices per order | Operational product reporting |
| **Gross (Items + Freight)** | Item prices + freight charges | Fulfillment cost analysis |
| **Paid Revenue** | Total captured payment per order | Finance reconciliation |
| **Net Revenue (Proxy)** | Paid revenue excluding canceled orders | Marketing attribution, ROI |

## Engineering Approach

The notebook follows a deliberate sequence designed to match production data pipeline standards:

**1. Schema validation before any computation**
Required columns are asserted before loading data. Missing files raise explicit errors with diagnostic messages.

**2. Grain checks before any joins**
Row counts per `order_id` are verified in each table before merging. Multi-item and multi-payment orders are identified and accounted for before they can cause problems downstream.

**3. Monetary sanity assertions**
Negative prices and payment values are flagged and blocked before they propagate into revenue calculations.

**4. Explicit order-level aggregation**
Item and payment tables are aggregated to order-level grain independently, then merged to the orders table using `validate="one_to_one"` to catch any grain violations at merge time.

**5. Reconciliation flags and deltas**
Canceled orders, orders without items, and orders without payments are flagged explicitly. Revenue deltas between definitions are computed and bucketed by root cause.

**6. Production validation checks**
Operational checks are exported as a standalone CSV, ready to be embedded in a production monitoring workflow.

## Key Findings

- Item-only revenue **understates customer-paid value by ~15%**, a systematic gap that accumulates at scale
- Items + freight revenue aligns within **~1% of paid revenue**, close enough to serve as a gross proxy
- Most discrepancies trace to **four root causes**: canceled orders with captured payments, missing item records, multi-item join inflation, and definition mismatch between teams
- After applying consistent order-level aggregation, the residual gap is explainable and expected, not a data error

## Mismatch Buckets

Each order is assigned to exactly one bucket using a priority hierarchy. An order qualifying for multiple conditions is assigned to the highest-priority bucket only.

| Bucket | Count | Description |
|---|---|---|
| `match_within_tolerance` | 98,405 | Revenue definitions align within rounding tolerance |
| `payment_no_items` | 614 | Orders with captured payments but no item records (non-canceled) |
| `items_freight_payment_delta_remaining` | 258 | Residual delta after accounting for all known root causes |
| `canceled_with_payment` | 163 | Canceled orders with captured payment (includes 161 that also have no items) |
| `items_no_payment` | 1 | Orders with items but no payment record |

**Note on bucket vs. validation check counts:** The standalone `validation_checks.csv` reports 775 orders without items total. The `mismatch_bucket` shows 614 `payment_no_items`. The difference of 161 orders are canceled orders that also have no items. These are prioritized into the `canceled_with_payment` bucket. The counts reconcile: 614 + 161 = 775. ✓

## Coverage & Data Quality

| Check | Severity | Count |
|---|---|---|
| Orders without item records | High | 775 |
| Orders without payment records | High | 1 |
| Payments referencing unknown orders | Critical | 0 |
| Items referencing unknown orders | Critical | 0 |
| Canceled orders with captured payment | Medium | 622 |
| Multi-payment orders | Low | 2,961 |
| Multi-item orders | Low | 9,803 |

The 775 orders without items are a known characteristic of the Olist dataset. They represent orders in early lifecycle states (created, invoiced, processing) where item records may not yet be fully populated. They are flagged and excluded from revenue totals rather than silently dropped.

## Known Limitations

- **`item_rows` and `payment_rows` stored as float64** rather than int, a result of NaN-filling during left joins before casting. This does not affect revenue calculations but would be corrected to int in a production pipeline.
- **Net revenue is a proxy only.** Refund data is not available in this dataset. Canceled orders are excluded as a conservative approximation; actual net revenue in a live system would require explicit refund records.
- **160 orders have no approval timestamp.** These are excluded from approval delay analysis but retained in all revenue calculations.

## Deliverables

| File | Description |
|---|---|
| `revenue_reconciliation_order_level.ipynb` | Full analysis, grain checks, reconciliation logic, and validation design |
| `data/processed/reconciliation_order_level.csv` | Final order-level reconciliation table (99,441 orders x 21 columns) |
| `docs/validation_checks.csv` | Proposed operational data quality checks for production monitoring |

## Takeaway

Revenue disagreements across teams are almost always caused by implicit assumptions about data structure and metric definitions, not bad data. The fix is not better dashboards. It is explicit grain control, documented metric logic, and automated validation checks that surface discrepancies before they reach stakeholders.

*Part of a portfolio focused on analytics engineering, data quality, and production-grade reporting systems.*

*Author: [Kablan Assebian](https://www.linkedin.com/in/gomis-kablan/) · [GitHub](https://github.com/Kablan-ASBN)*
