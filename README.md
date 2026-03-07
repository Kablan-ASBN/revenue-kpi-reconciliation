# Bank Loan Risk Analysis
### Default Risk Prediction on 1.2M+ Lending Club Loans

![Python](https://img.shields.io/badge/Python-3.10+-3776AB?style=flat&logo=python&logoColor=white)
![Pandas](https://img.shields.io/badge/Pandas-Data%20Wrangling-150458?style=flat&logo=pandas)
![scikit-learn](https://img.shields.io/badge/scikit--learn-Modeling-F7931E?style=flat&logo=scikit-learn)
![Status](https://img.shields.io/badge/Status-Complete-2ea44f?style=flat)

## The Problem

Default risk models trained on imbalanced data produce misleadingly high accuracy while failing to identify the cases that matter most. A model that predicts "fully paid" for every loan achieves 80% accuracy on Lending Club data, but catches almost none of the actual defaults. In lending, missing a default is far more costly than a false alarm.

This project investigates that tradeoff directly: what does it cost to optimize for accuracy, and what do you gain by optimizing for recall instead?

## Dataset

**Source:** [Lending Club Loan Data](https://www.kaggle.com/datasets/wordsforthewise/lending-club) — available on Kaggle.

The raw dataset contains 2,260,701 rows and 151 columns. After filtering to loans with known outcomes (Fully Paid or Charged Off) and selecting 16 relevant features, the working dataset is 1,265,976 rows.

**Target variable:** `loan_default` (binary: 1 = Charged Off, 0 = Fully Paid)

**Features used:**

| Type | Features |
|---|---|
| Numerical | `loan_amnt`, `int_rate`, `installment`, `annual_inc`, `dti`, `fico_range_high`, `open_acc`, `revol_util`, `total_acc` |
| Categorical | `term`, `emp_length`, `grade`, `home_ownership`, `purpose` |

## Pipeline Overview

**1. Data cleaning and feature engineering**
Filtered out in-progress loans to keep only resolved outcomes. Parsed mixed-type fields: `term` from "36 months" to numeric, `emp_length` from "10+ years" to float, `revol_util` from percentage string to float. Dropped rows with remaining NaN values. Final shape: 1,265,976 rows, 16 columns.

**2. Exploratory data analysis**
Boxplots comparing numeric distributions by default status, countplots for categorical features, and a correlation matrix to identify multicollinearity before modeling.

**3. Encoding**
One-hot encoded categorical variables using `pd.get_dummies()` with `drop_first=True`. Final feature matrix: 37 columns.

**4. Baseline model (imbalanced)**
Logistic regression via statsmodels trained on the full dataset and evaluated on the same data. Intercept added manually.

**5. Balanced model (downsampled)**
Downsampled the majority class to match the minority class size (247K each) using `sklearn.utils.resample()`. Retrained logistic regression and compared performance.

## EDA Highlights

**Boxplots: Numerical features vs loan default**

![Boxplots](outputs/Boxplots%20(numerical%20vs.%20loan_default).png)

Defaulters show higher interest rates and lower FICO scores on average. `dti`, `revol_util`, and `annual_inc` all exhibit heavy right tails driven by outliers.

**Countplots: Categorical features vs loan default**

![Countplots](outputs/Countplots%20(categorical%20vs.%20loan_default).png)

Grades B and C account for the highest loan volumes and the most fully paid loans overall. Grade A shows the best repayment ratio relative to its volume. Grades D, E, and F carry progressively worse default rates relative to their volume, representing the clearest risk signal across grade categories. Grade G has negligible volume.

Among home ownership types, MORTGAGE and RENT dominate total volume. RENT shows a slightly worse default-to-repayment ratio than MORTGAGE. OWN borrowers repay at a relatively higher rate.

Among loan purposes, `debt_consolidation` dominates total volume with a proportional but not alarming default rate. `small_business` stands out with a disproportionately high default rate relative to its volume. Low-volume categories like `renewable_energy`, `medical`, and `vacation` have insufficient observations to support reliable conclusions.

**Correlation matrix**

![Correlation Heatmap](outputs/Correlation%20heatmap.png)

`loan_amnt` and `installment` are highly correlated (0.95), indicating redundancy. The target variable shows weak individual correlations with all features (below 0.3), which is typical in credit risk and explains why logistic regression alone is limited here.

## Model Results

| Metric | Imbalanced Model | Balanced Model |
|---|---|---|
| Recall (defaults) | 5.4% | 67.1% |
| F1 score (defaults) | 0.098 | 0.658 |
| AUC | 0.710 | 0.7105 |
| Overall accuracy | 80.7% | 65.2% |

**Imbalanced ROC curve**

![ROC Imbalanced](outputs/ROC%20Curve%20for%20Loan%20Default%20Prediction.png)

**Balanced ROC curve**

![ROC Balanced](outputs/ROC%20Curve%20on%20Downsampled%20data.png)

The two AUC scores are nearly identical (0.710 vs 0.7105), which means the model's underlying discriminatory power did not change. What changed is the decision threshold behavior. The balanced model surfaces far more true defaults at any given threshold, which is the operationally correct objective for a risk team.

## Business Interpretation

The imbalanced model achieves 80.7% accuracy by predicting "fully paid" for nearly everything. It catches 5.4% of actual defaults. That is not a risk model — it is an accuracy illusion.

The balanced model drops to 65.2% accuracy but catches 67.1% of defaults. For a lending institution, the cost of a missed default (loan loss) is far higher than the cost of a false positive (declined good borrower). Optimizing for recall is the correct business decision, even at the expense of overall accuracy.

## Known Limitations

- **No train/test split.** Both models were trained and evaluated on the same data. Reported metrics reflect training performance, not generalization. A proper implementation would split the data before training and evaluate on a held-out test set only.
- **Logistic regression is a baseline, not a ceiling.** Gradient boosted models (LightGBM, XGBoost) would likely produce higher AUC and better recall with the same data. This project intentionally focuses on interpretability and business framing rather than performance optimization.
- **Downsampling discards data.** SMOTE or class weighting (`class_weight='balanced'`) would retain the full dataset and are worth comparing in a follow-up.

## Reproducing This Project

The raw data is not included in this repository due to file size. To reproduce:

1. Download the Lending Club dataset from [Kaggle](https://www.kaggle.com/datasets/wordsforthewise/lending-club)
2. Place the CSV at `data/bank_loan.csv`
3. Run `bank_loan_risk_analysis.ipynb` end to end

All outputs (plots, model results) are included in the `outputs/` folder.

## Takeaway

Accuracy is a misleading metric for imbalanced classification problems. In credit risk, the relevant question is not "how often is the model right?" but "how many defaults does it catch?" Reframing the evaluation criteria changes the model selection decision entirely.

*Part of a portfolio focused on analytics engineering, data quality, and production-grade reporting systems.*

*Author: [Kablan Assebian](https://www.linkedin.com/in/gomis-kablan/) · [GitHub](https://github.com/Kablan-ASBN)*
