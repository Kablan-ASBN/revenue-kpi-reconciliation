# Revenue & KPI Reconciliation  
**Order-Level Commerce Analytics Case Study**

## Overview
In practice, Finance, Operations, and Marketing teams often report different
revenue numbers from the same transaction data. These differences usually come
from **how revenue is defined and aggregated**, not from bad data.

This project walks through a full order-level reconciliation to show:
- why revenue numbers diverge,
- which definitions make sense for different business use cases, and
- what validation checks should exist to prevent confusion downstream.

The focus is on **data structure, definitions, and operational reliability** rather
than modeling or prediction.

---

## Business Questions
This analysis was driven by a few concrete questions:

1. Why do revenue totals differ across Finance and Operations reports?
2. How do item prices, freight charges, and payments interact at the order level?
3. Which revenue definition should be used for commerce reporting vs marketing attribution?
4. What data quality checks are necessary in a production environment?

---

## Data
**Source:** Brazilian E-Commerce Public Dataset (Olist)

**Tables used:**
- `orders` – order lifecycle, timestamps, and status  
- `order_items` – item-level prices and freight charges  
- `order_payments` – customer payment records  
- `customers` – customer identifiers (used for integrity checks)

All tables are explicitly aggregated to the **order level** before reconciliation.
No implicit joins or mixed grains are used.

---

## Revenue Definitions
Several revenue definitions are compared side-by-side on purpose:

- **Gross Revenue (Items Only)**  
  Sum of item prices per order. Freight excluded.

- **Gross Revenue (Items + Freight)**  
  Item prices plus freight charges.

- **Paid Revenue (All)**  
  Total captured customer payments per order.

- **Net Revenue (Proxy)**  
  Paid revenue excluding canceled orders.  
  *(Refund data is not available, so this is a conservative proxy.)*

---

## Key Findings
- Item-only revenue understates customer-paid value by roughly **15%**
- Including freight brings gross revenue within **~1%** of paid revenue
- A small but meaningful set of orders are **canceled after payment capture**
- Most discrepancies come from **definition choice and aggregation grain**, not data errors

---

## Coverage & Data Quality Observations
- ~99.2% of orders have item records
- ~99.99% of orders have payment records
- ~0.63% of orders are canceled
- Multi-item and multi-payment orders are common and expected

These patterns reinforce the need for explicit aggregation logic.

---

## Where Mismatches Come From
Once aggregated correctly, mismatches fall into a small number of buckets:

- Orders with payments but no items
- Orders with items but no payments
- Canceled orders with captured payments
- Legitimate multi-component orders (items, freight, split payments)

After applying consistent order-level definitions, most discrepancies disappear.

---

## Implications for Commerce & Marketing Analytics
- Using item-only revenue will systematically understate performance
- Marketing attribution should be based on **paid revenue adjusted for cancellations**
- Inconsistent timestamps can distort campaign effectiveness and ROI analysis

---

## Implications for Data Operations
These behaviors are normal in real commerce systems, but they must be monitored.

**Recommended validation checks:**
- Orders without items
- Orders without payments
- Canceled orders with payments
- Orders with multiple payment rows
- Orders with multiple item rows

Automating these checks improves reporting reliability and builds stakeholder trust.

---

## Deliverables
- `notebooks/revenue_reconciliation_order_level.ipynb`  
  Full analysis and reconciliation logic

- `data/processed/reconciliation_order_level.csv`  
  Final order-level reconciliation table

- `docs/validation_checks.csv`  
  Proposed operational data quality checks

---

## Takeaway
Revenue disagreements across teams are usually caused by **implicit assumptions**
about data structure and metric definitions.

Clear definitions, correct aggregation grain, and routine validation checks are
what make commerce analytics reliable at scale.
