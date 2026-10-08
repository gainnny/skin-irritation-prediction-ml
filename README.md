# Machine Learning-Based Skin Sensitization Prediction

This project develops and evaluates a Random Forest classifier for the binary skin-sensitization labels curated from the NTP ICE in vivo benchmark. Skin irritation and eye irritation are kept as separate auxiliary cohorts and are scored only after the sensitization model is fitted. They do not enter sensitization training, cross-validation, model selection, feature selection, or threshold selection.

## Data lineage and cohort curation

The primary source is `skin_sensitization.xlsx`, sheet `Data_invivo`. The data pipeline maps `Active` / `Sensitizer` to 1 and `Inactive` / `Non-sensitizer` to 0. In this source, the mapped responses occur under the `Call` and `EPA Classification` endpoint values. Standardized SMILES are used to remove conflicting-label structures and collapse duplicate structures before splitting.

| Sensitization curation stage | Records |
|---|---:|
| Raw `Data_invivo` rows | 13,945 |
| Responses accepted by the binary mapping | 3,256 (857 positive, 2,399 negative) |
| Valid standardized structures | 2,849 |
| Curated unique structures used for modeling | 1,185 (146 positive, 1,039 negative) |

The final cohort contains one row per standardized structure. Source workbooks and processed source datasets are not included in this repository. The code and final results are provided; reproduction requires access to the original workbooks placed locally under `data/raw/`.

## Features and representations

The classifier uses only these 10 RDKit descriptors: `MolWt`, `MolLogP`, `TPSA`, `NumHDonors`, `NumHAcceptors`, `NumRotatableBonds`, `HeavyAtomCount`, `RingCount`, `NumAromaticRings`, and `FractionCSP3`.

Radius-2 Morgan fingerprints (1,024 bits) and MACCS keys are also generated and exported as separate representations; neither is used as a classifier input. PCA, t-SNE, and UMAP use six physicochemical descriptors (`MolWt`, `MolLogP`, `TPSA`, `NumHDonors`, `NumHAcceptors`, `NumRotatableBonds`), not fingerprints.

## Model and evaluation

A stratified 80/20 split (`random_state=42`) gives 948 training records (117 positive, 831 negative) and 237 held-out test records (29 positive, 208 negative). Five-fold shuffled `StratifiedKFold` (`random_state=42`) is used on the training partition.

For every model comparison and every Random Forest grid-search fold, the complete estimator is a scikit-learn pipeline: `VarianceThreshold` -> `StandardScaler` -> `SelectKBest(mutual_info_classif, k=10)` -> classifier. The preprocessing steps are fitted separately on each CV training fold. The test data are transformed by the final training-fitted pipeline. The same 10-descriptor feature set and preprocessing are used throughout.

Six classifiers are compared: Random Forest, Extra Trees, CatBoost, SVM, Logistic Regression, and XGBoost. Random Forest hyperparameters are selected by five-fold `GridSearchCV` using the training partition and the existing search grid. The selected parameters were:

- `n_estimators`: 100
- `max_depth`: 10
- `min_samples_split`: 10
- `min_samples_leaf`: 4
- `max_features`: `sqrt`

The reported best grid-search CV ROC-AUC is **0.7852**. This is the selected score from the training CV search, not an independent validation estimate.

### Training-partition model comparison (five-fold CV)

| Model | Accuracy | Balanced accuracy | F1 | ROC-AUC |
|---|---:|---:|---:|---:|
| Random Forest | 0.8702 | 0.5584 | 0.2140 | 0.7843 |
| Extra Trees | 0.8576 | 0.5482 | 0.1831 | 0.7834 |
| CatBoost | 0.8692 | 0.5650 | 0.2272 | 0.7727 |
| SVM | 0.7775 | 0.6890 | 0.3859 | 0.7682 |
| Logistic Regression | 0.7469 | 0.7051 | 0.3861 | 0.7447 |
| XGBoost | 0.8660 | 0.5855 | 0.2754 | 0.7319 |

### Held-out test evaluation

The classification threshold is fixed at **0.50**. The test set is not used to choose the model or threshold. No test-based threshold optimization is performed.

| Metric | Held-out test |
|---|---:|
| Accuracy | 0.8439 |
| Balanced Accuracy | 0.6291 |
| F1-Score | 0.3509 |
| ROC-AUC | 0.7200 |
| Average Precision (PR-AUC) | 0.3366 |

All final metrics are also saved in `results/tables/`. Test features are used for post-fit SHAP and applicability-domain descriptions; test labels are not used for those analyses or for model/threshold selection.

## Auxiliary cross-endpoint analysis

The auxiliary cohorts are curated separately and are scored only after the sensitization pipeline is fitted. Skin irritation uses `skin_irritation.xlsx` / `Data_invivo`; its current curation is 200 structures (39 positive, 161 negative). The eye-irritation cohort uses `eye_irritation.xlsx` / `Data`; only `EPA Classification` and `GHS Classification` endpoint rows are included, yielding 333 structures (272 positive, 61 negative). Numerical Draize irritation scores, plateau levels, and intensity measurements are excluded from this classification cohort.

Together these are **533 endpoint records representing 388 unique standardized structures**, with **145 structures shared** between endpoint cohorts. The records are concatenated for post-fit probability profiling; the auxiliary labels are not used by the sensitization classifier. This analysis is descriptive cross-endpoint screening, not external validation of sensitization performance.

High- and low-probability subgroup summaries are in `results/tables/high_risk_vs_low_risk_descriptors.csv`; these are model-derived probability groups, not experimentally confirmed sensitization classes.

## Figures and result files

- `results/figures/`: regenerated curation, descriptor, CV, held-out test, SHAP, applicability-domain, and auxiliary-screening figures. No test-optimized threshold plot is included.
- `results/tables/`: cohort summaries, split counts, CV model comparison, grid-search summary, held-out test metrics, and auxiliary cohort/profile summaries.
- `models/sensitization_pipeline.joblib`: fitted preprocessing plus Random Forest pipeline; accepts the documented 10 raw descriptor columns.

## Limitations

- The source data are historical in vivo assay records and may contain assay and curation variability.
- The sensitization cohort is class-imbalanced and uses 10 two-dimensional descriptors.
- CV is used for model comparison and hyperparameter selection; the selected CV score is not an unbiased post-selection estimate.
- The held-out partition is an internal split from the curated cohort, not external validation.
- Auxiliary irritation probabilities are not evidence that the model predicts irritation or validates sensitization externally.
- Results do not establish clinical validity, causal mechanisms, or performance on unseen chemical scaffolds.


## Project scope

The notebooks implement data standardization, descriptor/fingerprint generation, sensitization model comparison and tuning, test evaluation, SHAP analysis, applicability-domain description, and post-fit auxiliary cohort scoring.
