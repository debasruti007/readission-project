# Healthcare Readmission Risk Prediction

Predicting 30-day hospital readmission for diabetic patients using the UCI **Diabetes 130-US Hospitals** dataset (101,766 encounters, 1999–2008). Built end-to-end: data cleaning → EDA → statistical testing → feature engineering → model comparison → hyperparameter tuning → threshold optimization → final risk-scored predictions.

## Results

| Model | ROC-AUC | PR-AUC | Notes |
|---|---|---|---|
| Baseline (majority class) | 0.500 | 0.112 | Always predicts "not readmitted" |
| Logistic Regression | 0.661 | 0.212 | Class-balanced |
| Random Forest | 0.665 | 0.219 | 300 trees, class-balanced |
| **CatBoost (final, tuned)** | **0.670** | **0.228** | Native categorical handling, tuned hyperparameters |

At the optimized decision threshold (0.54): **47% recall, 20% precision** on the readmit class. This is in line with published benchmarks on this dataset — the available fields are largely administrative/coded and don't fully capture the clinical complexity behind readmission, so this should be read as a *triage aid*, not an autonomous decision-maker. See [`Healthcare_Readmission_Report.docx`](./Healthcare_Readmission_Report.docx) for full methodology, limitations, and reasoning behind every modeling choice.

## What's in here

```
├── Healthcare_Readmission_FULL.ipynb   # Full executed notebook: data → EDA → modeling → final predictions
├── Healthcare_Readmission_Report.docx  # Full written report (methodology, results, limitations)
├── data/
│   ├── diabetic_data.csv               # Source data (101,766 encounters, 50 columns)
│   └── IDS_mapping.csv                 # Lookup tables for admission/discharge/source codes
└── outputs/
    ├── *.png                           # All generated plots (EDA, correlations, ROC/PR curves, feature importance, threshold sweep)
    ├── model_comparison.csv            # Baseline/LR/RF/CatBoost metrics
    ├── cv_results.csv                  # 3-fold cross-validation results
    ├── hyperparam_search.csv           # Hyperparameter search results
    ├── final_model_summary.json        # Final model config + test-set metrics
    └── readmission_predictions_final.csv   # Per-encounter predictions + risk tiers (Power BI-ready)
```

## Pipeline overview

1. **Data cleaning** — handled `?`-coded missing values (not blank cells, easy to miss), dropped two near-empty columns (`weight` 97% missing, `max_glu_serum` 95% missing), recoded the rest as `Unknown`/`Not Tested`, and excluded expired/hospice discharges (can't be meaningfully "readmitted").
2. **EDA & statistical testing** — readmission rate by demographic/clinical/administrative factor, correlation and VIF checks, chi-square and Mann-Whitney U tests to confirm associations before modeling.
3. **Feature engineering** — medication-change counts, total prior-utilization score, ICD-9 diagnosis grouping, rare-category collapsing.
4. **Patient-level train/test split** — `GroupShuffleSplit` on `patient_nbr` so no patient's encounters leak across train/test (this dataset has repeat visits per patient).
5. **Modeling** — Baseline, Logistic Regression, Random Forest, CatBoost; compared on ROC-AUC and **PR-AUC** (prioritized over accuracy given the ~11% positive class rate).
6. **Model improvement** — 3-fold patient-grouped cross-validation, threshold optimization (F1-maximizing cutoff instead of default 0.5), small hyperparameter search, feature re-selection (dropped 4 low-importance features with no performance cost).
7. **Final model** — CatBoost, evaluated once on the held-out test set to avoid overfitting the reported metric, exported with per-encounter risk tiers (Low/Medium/High).

## Key drivers of readmission risk

Based on CatBoost feature importance: **primary diagnosis group**, **total prior healthcare utilization** (past outpatient/emergency/inpatient visits), **age**, **number of diagnoses**, and **A1C test result**.

## Running it yourself

```bash
pip install pandas numpy matplotlib seaborn scipy scikit-learn statsmodels catboost
jupyter notebook Healthcare_Readmission_FULL.ipynb
# Run All — data paths are relative (./data/), so run from this folder
```

## Limitations

- Predictive power is modest (ROC-AUC ~0.67) — a known ceiling for this dataset given the available fields.
- Precision is low at the operating threshold (~1 in 5 flagged patients is a true positive); best used to narrow a large population for human review, not for automated action.
- Data is from 1999–2008 and may not reflect current clinical practice without revalidation.

Full discussion in the [report](./Healthcare_Readmission_Report.docx).

## Next steps

- Build a Power BI dashboard from `readmission_predictions_final.csv` (risk tiers, demographic breakdowns, driver features).
- Retrain with richer data if available (lab trends over time, social determinants, post-discharge follow-up compliance).
- Revalidate against current-era data before any real-world use.
