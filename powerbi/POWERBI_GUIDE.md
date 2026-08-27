# Power BI Build Guide — Readmission Risk Dashboard

Data file to use: `Power_BI_dataset.csv` (99,343 rows, cleaned and ready — no
nulls, no fixes needed). Don't use the earlier `readmission_predictions_final.csv`;
this one has the mapping gaps patched and a few extra display-friendly columns.

---

## 1. Import the data

1. Power BI Desktop → **Get Data → Text/CSV** → select `Power_BI_dataset.csv`.
2. In the preview window, click **Transform Data** (not Load) so you land in
   Power Query first.
3. Check column types: `encounter_id`, `patient_nbr` should be **Whole Number**;
   `time_in_hospital`, `num_medications`, `number_diagnoses` should be **Whole
   Number**; `readmission_probability` and `readmission_probability_pct` should
   be **Decimal Number**; `actual_readmission`, `predicted_readmission`,
   `correct_prediction` should be **True/False** or **Whole Number** (either
   works, but pick one and be consistent — DAX below assumes Whole Number 0/1).
4. Click **Close & Apply**.

There's only one table — no relationships to build. This is a flat,
one-row-per-encounter fact table, which keeps the model simple.

---

## 2. Fix age-group sort order (small but easy to miss)

The `age` column has values like `[0-10)`, `[10-20)` … `[90-100)`. Power BI
will sort these alphabetically by default, which happens to work correctly
here (each bucket's leading digit differs), so **no fix is actually required**
— but double check any chart using `age` on an axis shows buckets in the
right order after you build it. If it ever looks wrong, right-click the `age`
column → **Sort by column** and create a small index column in Power Query
(Add Column → Index) as the sort key.

---

## 3. DAX measures — copy-paste these

Create a new table for measures first (cleaner than scattering them on the
fact table): **Modeling → New Table**, name it `_Measures`, formula: `_Measures = ROW("x", 1)`.
Then add each measure below via **New Measure** with `_Measures` selected.

```dax
Total Encounters = COUNTROWS('Power_BI_dataset')

Actual Readmissions = SUM('Power_BI_dataset'[actual_readmission])

Actual Readmit Rate =
DIVIDE([Actual Readmissions], [Total Encounters], 0)

Predicted Readmissions = SUM('Power_BI_dataset'[predicted_readmission])

Predicted Flag Rate =
DIVIDE([Predicted Readmissions], [Total Encounters], 0)

Avg Readmission Probability =
AVERAGE('Power_BI_dataset'[readmission_probability])

High Risk Count =
CALCULATE([Total Encounters], 'Power_BI_dataset'[risk_level] = "High")

Medium Risk Count =
CALCULATE([Total Encounters], 'Power_BI_dataset'[risk_level] = "Medium")

Low Risk Count =
CALCULATE([Total Encounters], 'Power_BI_dataset'[risk_level] = "Low")

True Positives =
CALCULATE(
    [Total Encounters],
    'Power_BI_dataset'[actual_readmission] = 1,
    'Power_BI_dataset'[predicted_readmission] = 1
)

False Positives =
CALCULATE(
    [Total Encounters],
    'Power_BI_dataset'[actual_readmission] = 0,
    'Power_BI_dataset'[predicted_readmission] = 1
)

False Negatives =
CALCULATE(
    [Total Encounters],
    'Power_BI_dataset'[actual_readmission] = 1,
    'Power_BI_dataset'[predicted_readmission] = 0
)

True Negatives =
CALCULATE(
    [Total Encounters],
    'Power_BI_dataset'[actual_readmission] = 0,
    'Power_BI_dataset'[predicted_readmission] = 0
)

Model Precision =
DIVIDE([True Positives], [True Positives] + [False Positives], 0)

Model Recall =
DIVIDE([True Positives], [True Positives] + [False Negatives], 0)

Model F1 =
DIVIDE(
    2 * [Model Precision] * [Model Recall],
    [Model Precision] + [Model Recall],
    0
)

Model Accuracy =
DIVIDE([True Positives] + [True Negatives], [Total Encounters], 0)
```

These recompute live as slicers (age, gender, race, risk tier, admission
type…) are applied — e.g. selecting "High risk" in a slicer will recalculate
precision/recall for just that subset, which is genuinely useful for
exploring where the model over- or under-performs.

---

## 4. Dashboard pages — build these three

### Page 1 — Overview
- Four **Card** visuals across the top: `[Total Encounters]`, `[Actual Readmit
  Rate]` (format as %), `[High Risk Count]`, `[Model Recall]` (format as %).
- **Clustered bar chart**: Axis = `age`, Value = `[Actual Readmit Rate]`.
- **Donut chart**: Legend = `risk_level`, Value = `[Total Encounters]`.
- **Bar chart**: use the CatBoost feature-importance values from
  `../outputs/08_feature_importance.png` as a manually-entered small table
  (Power BI can't read CatBoost's internal importances directly — enter the
  top 5-8 feature names and importance scores from that chart as a new table
  via **Enter Data**), Axis = feature name, Value = importance.

### Page 2 — Risk drill-down
- **Slicers** along the left: `risk_level`, `age`, `gender`, `race`,
  `admission_type`.
- **Table or matrix**: Rows = `discharge_disposition_grp`-style grouping (or
  just `discharge_disposition` if you didn't collapse it), Values =
  `[Total Encounters]`, `[Actual Readmit Rate]`, `[Predicted Flag Rate]`.
- **Scatter chart**: X = `number_diagnoses`, Y = `readmission_probability`,
  Legend = `risk_level` — shows how probability relates to diagnosis count
  across risk tiers.

### Page 3 — Model performance
- Four cards: `[Model Precision]`, `[Model Recall]`, `[Model F1]`, `[Model
  Accuracy]`.
- **Table**: a manually-built 2x2 confusion matrix using `[True Positives]`,
  `[False Positives]`, `[False Negatives]`, `[True Negatives]` as four cards
  or a small matrix visual.
- **Line chart**: import `../outputs/09_threshold_sweep.png` data — if you
  want this interactive rather than a static image, re-export the
  `thr_df` table from the notebook (precision/recall/F1 by threshold) as a
  CSV and load it as a second table in Power BI; otherwise just insert the
  PNG as an image visual.

---

## 5. Publish to Power BI Service

1. In Power BI Desktop: **Home → Publish**.
2. Sign in with your Microsoft/organizational account if prompted.
3. Choose a workspace (My Workspace is fine for a personal/portfolio project;
   use a named workspace if this is for a team or class submission).
4. Once published, go to **app.powerbi.com**, open the workspace, and you'll
   see the report listed — click it to view/interact with it in the browser.
5. To share: **File → Publish to web** (public link, anyone can view — fine
   for a portfolio piece, not for anything with real patient identifiers) or
   use **Share** within the workspace to grant access to specific people
   (keeps it private, needs their Microsoft account).
6. Optional: **Pin visuals to a Dashboard** (a single-page summary view) from
   within the Service if you want a simplified landing page separate from the
   full multi-page report.

Note: `encounter_id` and `patient_nbr` in this export are the dataset's
original synthetic IDs, not real patient identifiers, so there's no PHI
concern with this specific dataset — but if you ever repeat this workflow on
real hospital data, publish to a private workspace only, never "Publish to
web".
