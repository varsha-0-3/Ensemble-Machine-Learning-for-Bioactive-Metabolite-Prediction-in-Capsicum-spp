# Ensemble Machine Learning for Bioactive Metabolite Prediction in *Capsicum* spp.

**R. Varsha Bantia**<sup>1</sup>, **Ashwani Sharma**<sup>2</sup>, **Anala M. R.**<sup>1</sup> (Member, IEEE), **Nagashree N. Rao**<sup>2</sup>, **Arushi Kadam**<sup>2</sup>, **Archith Shankar**<sup>2</sup>, and **K. N. Subramanya**<sup>3</sup>

<sup>1</sup> Department of Information Science and Engineering, RV College of Engineering, Bengaluru 560059, India  
<sup>2</sup> Department of Biotechnology, RV College of Engineering, Bengaluru 560059, India  
<sup>3</sup> Department of Industrial Engineering and Management, RV College of Engineering, Bengaluru 560059, India  

**Corresponding author:** Anala M. R. (analamr@rvce.edu.in)

This repository accompanies our manuscript of the same title. It contains the cleaned experimental datasets and the analysis notebook used to predict concentration of three bioactive metabolites — capsaicin, vanillylamine, and dihydrocapsaicin — from plant-growth-regulator (PGR) treatments in *Capsicum* spp. tissue culture.

HPLC remains the reference assay for these compounds, but it is slow and difficult to scale across many treatment combinations. We asked whether ensemble models, trained only on the concentrations of three regulators, could recover a useful share of the variation in metabolite yield and, in doing so, reduce the number of HPLC runs needed during formulation screening.

---

## What this repository contains

| Path | Description |
| --- | --- |
| `capsicum.ipynb` | End-to-end analysis. Three independent pipelines: capsaicin, vanillylamine, and dihydrocapsaicin. |
| `data/` | Standardised modelling tables and the companion raw tables used for feature scatter plots. |
| `requirements.txt` | Python packages required to rerun the notebook. |
| `figures/` | Created when the notebook is run. Not required as an input. |

The notebook reads files from the **working directory**, not from `data/`. Before running it, either copy the data files into the same folder as the notebook, or change the `read_csv` / `read_excel` paths. The names below are the names the code actually opens.

| Compound | Modelling table (standardised) | Companion table (raw scale, scatter plots) |
| --- | --- | --- |
| Capsaicin | `Cap_os_standardized_data_1_.csv` | `Capsaicin_changed_1_.csv` |
| Vanillylamine | `Vanilylamine_standardized_data.xlsx` | `Vanilylamine_changed.xlsx`, third sheet (`sheet_name=2`) |
| Dihydrocapsaicin | `DiCap_standardized_data.xlsx` | `DiCap_changed.xlsx`, third sheet (`sheet_name=2`) |

---

## Input-Output

For each metabolite we treat the problem as a separate regression:

- **Inputs:** TIBA (2,3,5-triiodobenzoic acid), TDZ (thidiazuron), and BA (benzyladenine), in mg/L.
- **Target:** mass-normalised yield, `nanogramPerG` (ng of compound per g of sample).

Yield is compound mass divided by sample weight so that observations from different sample sizes can be compared. Features and the target are standardised independently with `StandardScaler` (zero mean, unit variance, population standard deviation) **after** cleaning. Reported MAE, RMSE, and bias are therefore on the standardised scale and can be compared across compounds even though the raw concentration ranges differ.

Germplasm was obtained from IIHR, Bengaluru (Arka Lohit) and DRDO, Pithoragarh (KTOC). Fruits collected after anthesis were extracted and quantified by HPLC (isocratic acetonitrile:water 1:1, detection at 236 nm, 1 mL/min). The laboratory workflow is Figure 2 of the manuscript.

---

## Dataset after cleaning

Outliers and rows that were not original HPLC measurements were removed and are documented in the exclusion log supplied with the data. In particular:

- **Capsaicin:** one mean-imputed row and two linearly interpolated rows were excluded.
- **Vanillylamine:** eight rows were excluded, including two extreme yields and five modified HPLC values that could not be verified. Three interpolated observations had been added earlier, only inside the observed PGR range; vanillylamine results should be read with that augmentation in mind.
- **Dihydrocapsaicin:** two extreme outlier rows, one row flagged as very small, the lower of two technical replicates, and three synthetic rows were removed.

| Compound | n | TIBA (mg/L) | TDZ (mg/L) | BA (mg/L) | Yield (ng/g) |
| --- | ---: | --- | --- | --- | --- |
| Capsaicin | 44 | 0.00–1.00 | 0.00–4.00 | 0.00–8.00 | 251.3–13,748.3 |
| Vanillylamine | 31 | 0.00–1.00 | 0.00–3.00 | 0.00–4.00 | 156.3–24,908.4 |
| Dihydrocapsaicin | 32 | 0.00–1.00 | 0.00–4.00 | 0.00–4.00 | 222.4–15,616.3 |

Each modelling table must contain the columns `TIBA`, `TDZ`, `BA`, and `nanogramPerG`. The notebook does not re-derive ng/g from sample weight; that normalisation is already in the released tables.

---

## Models and evaluation

We compared four tree ensembles — Random Forest, Gradient Boosting, XGBoost, and CatBoost — against a multiple linear regression (MLR) baseline. All stochastic models were initialised with `random_state=42`.

**Primary protocol.** Leave-one-out cross-validation (LOO CV). In each fold the model is refit from scratch on *n* − 1 samples and predicts the held-out sample. *R*<sup>2</sup>, MAE, and RMSE are computed on the resulting out-of-sample vector.

**Hyperparameter search.** `GridSearchCV` combined with LOO produced a NaN failure on these tables, so the notebook implements an explicit loop that is methodologically the same thing: every grid combination is scored by a full LOO pass, and the combination with the highest LOO *R*<sup>2</sup> is retained. Search sizes:

| Model | Combinations | Model fits, capsaicin (*n* = 44) | Vanillylamine (*n* = 31) | Dihydrocapsaicin (*n* = 32) |
| --- | ---: | ---: | ---: | ---: |
| Random Forest | 540 | 23,760 | 16,740 | 17,280 |
| Gradient Boosting | 384 | 16,896 | 11,904 | 12,288 |
| XGBoost | 972 | 42,768 | 30,132 | 31,104 |
| CatBoost | 192 | 8,448 | 5,952 | 6,144 |

A complete rerun is therefore hours of compute, not minutes. If you only want the published models, skip the search cells and instantiate the hyperparameters in the table below.

**Uncertainty.** Each LOO prediction vector was resampled with replacement 5,000 times. The 2.5th and 97.5th percentiles of the resulting *R*<sup>2</sup> distribution are the 95% confidence interval. Residual normality was assessed with the Shapiro–Wilk test. Mean bias is the mean of (actual − predicted) on the standardised scale.

**Secondary splits.** 90/10, 80/20, and 70/30 train–test splits (`random_state=42`) are reported for comparison. With these sample sizes they are not the primary evidence.

**Interpretation.** SHAP values (`TreeExplainer`) were computed for the best model of each compound, fit on the full cleaned table. Permutation importance (30 repeats) was computed for all four models as a model-agnostic check. Both analyses ranked **TIBA** first in all 12 model–compound combinations, which is consistent with its role as an inhibitor of polar auxin transport.

### Hyperparameters selected by LOO grid search

| Compound | Model | Trees / iterations | Learning rate | Depth | Other |
| --- | --- | ---: | ---: | --- | --- |
| Capsaicin | Random Forest | 100 | — | 7 | `min_samples_split=2`, `min_samples_leaf=1`, `max_features=sqrt` |
| Capsaicin | Gradient Boosting | 200 | 0.01 | 3 | `subsample=1.0`, `min_samples_split=2` |
| Capsaicin | XGBoost | 200 | 0.01 | 3 | `min_child_weight=1`, `subsample=1.0`, `colsample_bytree=1.0`, `reg_alpha=0` |
| Capsaicin | CatBoost | 2000 | 0.01 | 8 | `l2_leaf_reg=5` |
| Vanillylamine | Random Forest | 500 | — | 5 | `min_samples_split=2`, `min_samples_leaf=1`, `max_features=None` |
| Vanillylamine | Gradient Boosting | 200 | 0.05 | 2 | `subsample=1.0`, `min_samples_split=2` |
| Vanillylamine | XGBoost | 100 | 0.10 | 3 | `min_child_weight=1`, `subsample=1.0`, `colsample_bytree=0.7`, `reg_alpha=0.1` |
| Vanillylamine | CatBoost | 500 | 0.10 | 4 | `l2_leaf_reg=3` |
| Dihydrocapsaicin | Random Forest | 300 | — | None | `min_samples_split=2`, `min_samples_leaf=1`, `max_features=sqrt` |
| Dihydrocapsaicin | Gradient Boosting | 100 | 0.05 | 2 | `subsample=1.0`, `min_samples_split=2` |
| Dihydrocapsaicin | XGBoost | 500 | 0.01 | 3 | `min_child_weight=5`, `subsample=1.0`, `colsample_bytree=0.7`, `reg_alpha=0.5` |
| Dihydrocapsaicin | CatBoost | 500 | 0.03 | 8 | `l2_leaf_reg=7` |

---

## Principal results

MLR under the same LOO protocol failed: LOO *R*<sup>2</sup> was −0.106 for capsaicin, −0.247 for vanillylamine, and 0.022 for dihydrocapsaicin. The PGR–yield relationship is not linear. The best ensemble improved on MLR by +0.65, +0.88, and +0.52 absolute *R*<sup>2</sup> points, respectively.

| Compound | *n* | Best model | LOO *R*<sup>2</sup> | 95% CI | MAE | RMSE |
| --- | ---: | --- | ---: | --- | ---: | ---: |
| Capsaicin | 44 | CatBoost | 0.5469 | [0.111, 0.758] | 0.475 | 0.673 |
| Vanillylamine | 31 | XGBoost | 0.6351 | [0.207, 0.796] | 0.457 | 0.604 |
| Dihydrocapsaicin | 32 | CatBoost | 0.5409 | [0.199, 0.735] | 0.537 | 0.678 |

Full LOO metrics (manuscript Tables 6–8):

**Capsaicin (*n* = 44)**

| Model | LOO *R*<sup>2</sup> | 95% CI | MAE | RMSE | Shapiro–Wilk *p* | Bias |
| --- | ---: | --- | ---: | ---: | ---: | ---: |
| Random Forest | 0.4880 | [0.042, 0.695] | 0.595 | 0.716 | 0.073 | −0.055 |
| Gradient Boosting | 0.4839 | [0.002, 0.700] | 0.600 | 0.718 | 0.124 | −0.043 |
| XGBoost | 0.4382 | [−0.063, 0.660] | 0.640 | 0.750 | 0.063 | −0.048 |
| CatBoost | 0.5469 | [0.111, 0.758] | 0.475 | 0.673 | 0.073 | −0.149 |

**Vanillylamine (*n* = 31)**

| Model | LOO *R*<sup>2</sup> | 95% CI | MAE | RMSE | Shapiro–Wilk *p* | Bias |
| --- | ---: | --- | ---: | ---: | ---: | ---: |
| Random Forest | 0.4030 | [−0.027, 0.596] | 0.604 | 0.773 | 0.431 | +0.018 |
| Gradient Boosting | 0.6210 | [0.253, 0.782] | 0.458 | 0.616 | 0.111 | +0.038 |
| XGBoost | 0.6351 | [0.207, 0.796] | 0.457 | 0.604 | 0.447 | +0.031 |
| CatBoost | 0.4673 | [0.018, 0.694] | 0.521 | 0.730 | 0.022 | +0.067 |

**Dihydrocapsaicin (*n* = 32)**

| Model | LOO *R*<sup>2</sup> | 95% CI | MAE | RMSE | Shapiro–Wilk *p* | Bias |
| --- | ---: | --- | ---: | ---: | ---: | ---: |
| Random Forest | 0.4812 | [0.136, 0.679] | 0.597 | 0.720 | 0.073 | +0.010 |
| Gradient Boosting | 0.4177 | [−0.033, 0.664] | 0.614 | 0.763 | 0.641 | −0.003 |
| XGBoost | 0.4687 | [0.112, 0.658] | 0.632 | 0.729 | 0.096 | +0.013 |
| CatBoost | 0.5409 | [0.199, 0.735] | 0.537 | 0.678 | 0.055 | −0.021 |

These *R*<sup>2</sup> values should be read against the biology, not against a large-sample benchmark. Capsaicinoid accumulation depends on genotype, culture duration, environment, and plant-to-plant noise as well as the PGR regime. Genomic-selection studies that use thousands of SNP markers have reported prediction accuracies of about 0.55–0.70 for capsaicinoid content. Reaching a similar range from three treatment variables, with no genetic features, is the result we consider worth reporting. The confidence intervals are wide because *n* is 31–44. That width is the uncertainty, not a defect to be hidden.

CatBoost residuals for vanillylamine departed mildly from normality (Shapiro–Wilk *p* = 0.022). CatBoost is not the selected model for that compound, so this does not change the primary conclusion.

---

## How to reproduce

The notebook was developed in Google Colab (Python 3). It also runs locally. Figures are written at 300 dpi with a serif face and the Agg backend.

```bash
python -m venv .venv
# Windows
.venv\Scripts\activate
# macOS / Linux
source .venv/bin/activate

pip install -r requirements.txt
