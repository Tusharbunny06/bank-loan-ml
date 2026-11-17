# Power BI: one-page Bank Loan Risk dashboard

This guide builds the dashboard from the files the notebook exports into `outputs/`. Follow it top to bottom; every step is a click path or a piece of code to paste.

## 0. What you need

| File (in `outputs/`) | Rows | Use in Power BI |
|---|---|---|
| `scored_customers.csv` | 32,411 | **Main table.** One row per customer with the ML risk score, risk band and segment. Almost every visual uses it. |
| `feature_importance.csv` | 23 | "Why the model says risky" bar chart (SHAP). |
| `model_comparison.csv` | 5 | Model quality cards (ROC-AUC, recall, precision). |
| `confusion_matrix.csv` | 4 | Caught vs missed defaults. |
| `segment_summary.csv` | 3 | Optional: segment profile table. |
| `bank_ml_theme.json` (in `powerbi/`) | - | Colour theme that matches the notebook charts. |

Use **Power BI Desktop** (free, Windows). Save your report as `Bank_Loan_Risk.pbix`.

## 1. Load the data

1. **Home > Get data > Text/CSV**, pick `scored_customers.csv`, click **Transform Data** (not Load).
2. In Power Query: **Home > Advanced Editor**, delete everything and paste the code below. **Change the file path on the first line** to where your file really is.

```m
let
    Source = Csv.Document(File.Contents("C:\path\to\bank_ml\outputs\scored_customers.csv"),
                [Delimiter=",", Columns=24, Encoding=65001, QuoteStyle=QuoteStyle.Csv]),
    Promoted = Table.PromoteHeaders(Source, [PromoteAllScalars=true]),
    Typed = Table.TransformColumnTypes(Promoted, {
        {"customer_id", Int64.Type}, {"person_age", Int64.Type}, {"age_band", type text},
        {"person_income", Int64.Type}, {"income_band", type text}, {"person_home_ownership", type text},
        {"person_emp_length", type number}, {"loan_intent", type text}, {"loan_grade", type text},
        {"loan_amnt", Int64.Type}, {"loan_amnt_band", type text}, {"loan_int_rate", type number},
        {"loan_percent_income", type number}, {"dti_band", type text}, {"cb_person_default_on_file", type text},
        {"cb_person_cred_hist_length", type number}, {"interest_burden", type number},
        {"segment_id", Int64.Type}, {"segment", type text}, {"loan_status", Int64.Type},
        {"default_probability", type number}, {"risk_band", type text}, {"predicted_default", Int64.Type},
        {"prediction_outcome", type text}}, "en-US"),
    // sort-order helper columns so bands appear Low -> High instead of alphabetically
    AddRiskOrder = Table.AddColumn(Typed, "risk_order", each List.PositionOf({"Low","Medium","High"}, [risk_band]) + 1, Int64.Type),
    AddAgeOrder = Table.AddColumn(AddRiskOrder, "age_order", each List.PositionOf({"25 and under","26 to 30","31 to 35","36 to 45","46 and over"}, [age_band]) + 1, Int64.Type),
    AddIncomeOrder = Table.AddColumn(AddAgeOrder, "income_order", each List.PositionOf({"Under 30K","30K to 50K","50K to 75K","75K to 100K","100K and over"}, [income_band]) + 1, Int64.Type),
    AddLoanOrder = Table.AddColumn(AddIncomeOrder, "loan_amnt_order", each List.PositionOf({"Under 5K","5K to 10K","10K to 15K","15K to 20K","20K and over"}, [loan_amnt_band]) + 1, Int64.Type),
    AddDtiOrder = Table.AddColumn(AddLoanOrder, "dti_order", each List.PositionOf({"Under 10%","10% to 20%","20% to 30%","Over 30%"}, [dti_band]) + 1, Int64.Type)
in
    AddDtiOrder
```

The final `"en-US"` makes decimals like `0.55` read correctly even if your Windows uses a comma as the decimal separator.

3. Click **Done**, then rename the query (right panel) to `scored_customers`.
4. Repeat **Get data > Text/CSV** for `feature_importance.csv`, `model_comparison.csv` and `confusion_matrix.csv` (these are tiny; plain **Load** is fine and the defaults are correct). If asked, use **Use first row as headers**.
5. **Home > Close & Apply**.

Do **not** create relationships between these tables. The small tables are static results of the model, so they intentionally ignore the slicers (see section 5).

## 2. Set sort orders (so bands show in a sensible order)

In **Data view**, click the column, then **Column tools > Sort by column**:

| Column | Sort by |
|---|---|
| `risk_band` | `risk_order` |
| `age_band` | `age_order` |
| `income_band` | `income_order` |
| `loan_amnt_band` | `loan_amnt_order` |
| `dti_band` | `dti_order` |

Then hide the helper columns (right-click > Hide in report view): `risk_order`, `age_order`, `income_order`, `loan_amnt_order`, `dti_order`.

## 3. DAX measures

Select the `scored_customers` table, then **Home > New measure** for each one. Paste one measure at a time.

```dax
Total Customers = DISTINCTCOUNT(scored_customers[customer_id])

Actual Default Rate = AVERAGE(scored_customers[loan_status])

Avg Predicted Risk = AVERAGE(scored_customers[default_probability])

High Risk Customers =
CALCULATE(COUNTROWS(scored_customers), scored_customers[risk_band] = "High")

High Risk % = DIVIDE([High Risk Customers], [Total Customers])

Avg Loan Amount = AVERAGE(scored_customers[loan_amnt])

Total Loan Exposure = SUM(scored_customers[loan_amnt])

Exposure in High Risk =
CALCULATE(SUM(scored_customers[loan_amnt]), scored_customers[risk_band] = "High")

Expected Loss Amount =
SUMX(scored_customers, scored_customers[loan_amnt] * scored_customers[default_probability])

Model ROC-AUC =
CALCULATE(MAX(model_comparison[ROC-AUC]), model_comparison[model] = "XGBoost (tuned)")

Model Recall =
CALCULATE(MAX(model_comparison[Recall]),
          model_comparison[model] = "XGBoost (tuned) @ cost threshold")
```

Format them with **Measure tools**: `Actual Default Rate`, `Avg Predicted Risk`, `High Risk %`, `Model Recall` as **Percentage (1 decimal)**; `Model ROC-AUC` as **Decimal (3)**; the exposure and loss measures as **Currency (no decimals)**.

`Expected Loss Amount` multiplies each loan by its predicted default probability. It is a rough "money at risk" figure for the selected customers, not an accounting loss.

## 4. Theme

**View > Themes > Browse for themes**, choose `powerbi/bank_ml_theme.json`. This gives the same blue/orange palette as the notebook and light card borders.

## 5. Page layout (one page, 16:9)

Use the default 16:9 canvas. Rough grid: 4 rows.

```
+--------------------------------------------------------------------------------+
| Row 1: Title (left)      Slicers: Segment | Risk band | Loan intent | Home own. |
+--------------------------------------------------------------------------------+
| Row 2: 6 KPI cards                                                             |
|  Customers | Default rate | High-risk % | Avg loan | $ in high risk | ROC-AUC   |
+--------------------------------------------------------------------------------+
| Row 3:  Risk band chart  |  Default rate by grade  |  Segment table             |
+--------------------------------------------------------------------------------+
| Row 4:  Default rate by loan-to-income | Top risk drivers | Caught vs missed     |
+--------------------------------------------------------------------------------+
```

### Row 1: title and slicers
- **Text box**: `Bank Loan Risk: Customer Segments and Default Prediction` (Segoe UI Semibold, 20 pt).
- **4 slicers** (Visualizations pane > Slicer), style **Dropdown** (Format > Slicer settings > Options > Style):
  - `scored_customers[segment]`
  - `scored_customers[risk_band]`
  - `scored_customers[loan_intent]`
  - `scored_customers[person_home_ownership]`

### Row 2: six cards (Visualizations pane > **Card**)
| Card | Field | Note |
|---|---|---|
| Customers | `Total Customers` | |
| Default rate | `Actual Default Rate` | Actual outcomes in the data |
| High-risk % | `High Risk %` | Share of customers the model bands as High |
| Avg loan | `Avg Loan Amount` | |
| Loan value in high risk | `Exposure in High Risk` | |
| Model ROC-AUC | `Model ROC-AUC` | Static, does not change with slicers |

### Row 3
1. **Customers by risk band**: *Clustered column chart*. X-axis = `risk_band`, Y-axis = `Total Customers`. Turn on data labels. Title: "Customers by ML risk band".
2. **Default rate by lender grade**: *Clustered column chart*. X-axis = `loan_grade`, Y-axis = `Actual Default Rate` (shown as %). Data labels on. Title: "Default rate by loan grade".
3. **Segment table**: *Table* visual with `segment`, `Total Customers`, `Actual Default Rate`, `Avg Predicted Risk`, `Avg Loan Amount`. Title: "Customer segments". Optional: conditional formatting on `Actual Default Rate` (Format > Cell elements > Background color > a blue-to-orange scale).

### Row 4
4. **Default rate by loan-to-income**: *Clustered column chart*. X-axis = `dti_band`, Y-axis = `Actual Default Rate`. Title: "Default rate by loan size vs income".
5. **Top risk drivers**: *Clustered bar chart* from `feature_importance`. Y-axis = `feature`, X-axis = `mean_abs_shap` (Sum). Filters pane: this visual > `feature` > **Top N**: Top 10 by `mean_abs_shap`. Title: "What drives default risk (SHAP)".
6. **Caught vs missed defaults**: *Matrix* visual from `confusion_matrix`. Rows = `actual`, Columns = `predicted`, Values = `count` (Sum). Title: "Model decisions on the test set".

### How slicers behave (explain this to your reviewer)
Slicers filter everything built from `scored_customers`. Visuals 5 and 6 and the ROC-AUC card come from the model-evaluation tables, which describe the **model**, not a customer subset, so they correctly stay fixed. To make that obvious, add a small note under those visuals: "Model results (test set), not affected by filters."

## 6. Final polish checklist
- Every chart has a title and data labels; axis titles only where the unit is not obvious.
- Rename fields in the visuals to plain English (double-click the field in the Visualizations pane).
- Align visuals with **Format > Align** and **Distribute**; leave equal gaps.
- Click each slicer value (e.g. Segment = High-Risk Borrowers) and confirm the cards and charts change sensibly.
- **View > Page view > Fit to page** and check nothing overflows.
- Add **alt text** to each visual (Format > General > Alt text) for accessibility.

## 7. Numbers to expect (sanity check)
With no slicers applied you should see roughly:
- Total customers: **32,411**
- Actual default rate: **about 21.9%**
- Risk bands: **Low about 21,000 (3% default), Medium about 5,600 (19%), High about 5,800 (92%)**
- Model ROC-AUC: **0.948**

If your numbers differ, re-check the file path, the `en-US` locale step, and that you loaded `outputs/scored_customers.csv` (not `cleaned_dataset.csv`).
