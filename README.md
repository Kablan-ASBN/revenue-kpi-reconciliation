# Revenue and KPI Reconciliation (Olist E-Commerce)

This project reconciles revenue metrics across order items and payments to explain why finance and analytics teams often report different numbers.

## What this answers
- Why paid revenue differs from item revenue
- How freight, cancellations, and multi-payment orders impact totals
- Which validation checks should run daily to maintain data quality

## Dataset
Brazilian E-Commerce Public Dataset by Olist (2016 to 2018).  
Raw data is not committed to the repo.

## Repo structure
- `revenue_reconciliation_order_level.ipynb` main analysis notebook
- `data/processed/` order-level reconciliation output
- `docs/validation_checks.csv` validation checks and counts

## Key outputs
- `data/processed/reconciliation_order_level.csv`
- `docs/validation_checks.csv`

## Notes
All metrics are computed at order grain to avoid double counting caused by item-level or payment-level joins.
