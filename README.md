# Banking Customer & Loan Default Analysis (ML Project)

A machine-learning project that answers two questions a bank asks:

1. **Will this applicant repay the loan?** (default prediction, supervised ML)
2. **What kinds of customers do we have?** (customer segmentation, unsupervised ML)

The outputs feed a one-page Power BI dashboard.

## Dataset

Credit Risk Dataset (Kaggle): 32,580 loan applicants, 12 columns. Target: `loan_status` (1 = defaulted, 0 = repaid).

## What the notebook covers

| Part | Content |
|---|---|
| 1. Load data | Column meanings and types, summary statistics |
| 2. Data quality audit | Missing values, duplicates, impossible values |
| 3. Cleaning | Duplicates removed, invalid values fixed, missing-value test plus 5 imputation methods compared, outliers (IQR and z-score) treated per column, log transform for skew |
| 4. Feature engineering | Risk ratios, business bands, encoding |
| 5. EDA | Distributions, default rate by group, correlation, Mann-Whitney and chi-square tests |
| 6. Segmentation | K-Means (elbow and silhouette), PCA view, 3 named customer segments |
| 7. Default prediction | Class-imbalance strategies, Logistic Regression vs Random Forest vs XGBoost, hyperparameter tuning, test-set evaluation, calibration, cost-based threshold, SHAP explanations |
| 8. Export | Every customer scored (out-of-fold), files exported for Power BI |
| 9. Summary | Key results and limitations |

## Key results (test set)

- Tuned XGBoost: **ROC-AUC 0.948**, **PR-AUC 0.906**
- At the cost-based threshold (0.15): precision 0.66, recall 0.86
- Business cost cut from 7,090 units (approve everyone) to 1,615 units
- Risk bands: Low 3.1%, Medium 19.0%, High 92.5% actual default rate

## Project structure

```
Bank_Customer_Loan_Analysis.ipynb   main notebook
data/credit_risk_dataset.csv        raw data
outputs/                            cleaned data, scored customers, model, metrics
outputs/figures/                    all 25 charts
powerbi/                            Power BI guide and colour theme
requirements.txt
```

## How to run

```bash
pip install -r requirements.txt
jupyter notebook Bank_Customer_Loan_Analysis.ipynb
```

Run it from this folder so the notebook can find `data/credit_risk_dataset.csv`. A full run takes about a minute.
