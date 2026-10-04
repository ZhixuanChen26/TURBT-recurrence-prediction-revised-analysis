# Two-year recurrence prediction

Multimodel comparison, calibration, decision-curve analysis, and SHAP explanation for recurrence within two years.

[!\[Python](https://img.shields.io/badge/Python-3.8%2B-3776AB)](https://www.python.org/)
\[!\[Status](https://img.shields.io/badge/status-research%20code-orange)]()

This repository predicts **recurrence within two years** (`Recurrence`: 0 = no recurrence, 1 = recurrence) from preoperative clinical, laboratory, pathological, and imaging variables. It publishes the analysis code and method notes only. **It does not contain patient-level data.**

\---

## What is included

|Included|Not included|
|-|-|
|Analysis notebook with outputs cleared|Raw or imputed Excel / CSV files|
|Method notes, dependencies, and random seeds|Medical-record numbers, names, examination dates, images|
|Column names and encodings required by the figure scripts|Any intermediate table that can reconstruct an individual|

The raw-table path in the notebook is a local example only. Change it to a relative path or an environment variable before upload, and exclude data files in `.gitignore`.

\---

## Analysis pipeline

The notebook is a research script meant to be run cell by cell. It is not an installable package.

1. **Load and clean**  
Read the Excel file with `header=2`, drop the Chinese description row and columns with more than 80% missingness, and remove the identifier `ID` and `Relapse interval`. Sex, smoking, and drinking are encoded from 男/女 and 是/否 to 0/1.
2. **Baseline table**  
Continuous variables are tested for normality. Normal variables are reported as mean ± standard deviation and compared with a t-test; otherwise median (interquartile range) and Mann–Whitney U are used, with a Z statistic. Categorical variables use the chi-square test, or Fisher's exact test for a 2×2 table with an expected count below 5. These comparisons are train versus test, not recurrence versus no recurrence.
3. **Split and impute (fit on the training set only)**  
`train\_test\_split(test\_size=0.3, stratify="Recurrence", random\_state=0)`.  
Continuous variables: `IterativeImputer(BayesianRidge, max\_iter=20, random\_state=42)`.  
Categorical variables: training-set mode. The test set is transformed only.
4. **Feature selection on the training set**

   * L1 logistic regression (`LogisticRegressionCV`, 5-fold, `neg\_log\_loss`, features standardized first): 29 nonzero coefficients.
   * Boruta with a random forest, confirmed set: 17 features.
   * Intersection, 10 features: `APTT`, `Depth of invasion`, `Pathological grade`, `RDW-CV`, `Radiologic necrosis`, `Smoking`, `TG`, `Tumor location`, `Urine RBC count`, `Urine specific gravity`.
   * **Final model matrix, 8 features** (the notebook manually drops `RDW-CV` and `Urine specific gravity`):

|Variable|Type|Meaning (encoding as in the notebook)|
|-|-|-|
|APTT|Continuous|Activated partial thromboplastin time|
|TG|Continuous|Triglycerides|
|Urine RBC count|Continuous|Urine red-cell count|
|Depth of invasion|Categorical|Depth of invasion|
|Pathological grade|Categorical|Pathological grade|
|Radiologic necrosis|Categorical|Radiologic necrosis, 0/1|
|Smoking|Categorical|Smoking, 0/1|
|Tumor location|Categorical|Tumor-site code|

5. **Six classifiers** (`SEED=180`). Logistic regression and SVM standardize continuous variables; tree models do not. CatBoost receives categorical columns directly.

|Model|Main settings|
|-|-|
|Logistic Regression|L2, `C=0.3`, `liblinear`|
|SVM|RBF, `C=100`, `gamma=0.03`; evaluation uses `decision\_function`, not probability|
|Random Forest|150 trees, `max\_depth=4`, `min\_samples\_leaf=5`, `balanced\_subsample`|
|XGBoost|150 trees, `max\_depth=4`, `learning\_rate=0.01`, `scale\_pos\_weight=2.3`|
|CatBoost|250 iterations, `depth=6`, `learning\_rate=0.01`, `auto\_class\_weights="Balanced"`|
|GBDT|100 trees, `max\_depth=4`, `learning\_rate=0.01`, `min\_samples\_leaf=15`|

   These are the final settings written into the notebook, not the result of a new search in this repository. The classification threshold is the Youden point of the **training** ROC, then applied to the test set.

6. **Test-set evaluation**  
AUROC, AUPRC, sensitivity, specificity, accuracy, F1, and related metrics, with 95% intervals from 1,000 stratified bootstrap resamples. AUC comparisons use the paired DeLong test, with Holm correction across all 15 pairwise tests. Later cells add sigmoid calibration (`CalibratedClassifierCV`, 5-fold, without a new train/test split), decision-curve analysis (thresholds 0.01–0.99), and native CatBoost SHAP values on the log-odds scale.

One local run produced the following test AUCs (DeLong normal-approximation 95% CI). Use them to check the pipeline, not as external validation:

|Model|Test AUC (DeLong 95% CI)|
|-|-|
|Logistic Regression|0.678 (0.549–0.807)|
|SVM|0.749 (0.638–0.861)|
|Random Forest|0.788 (0.688–0.888)|
|XGBoost|0.782 (0.678–0.886)|
|CatBoost|0.795 (0.699–0.892)|
|GBDT|0.769 (0.664–0.874)|

After Holm correction, only CatBoost and random forest differed from logistic regression at the 0.05 level. Differences among the tree models were not significant. The intervals are wide, and the number of test-set events is limited.

\---

## Environment

The notebook kernel was Python 3.8.```bash
python -m venv .venv
# Windows: .venv\\Scripts\\activate
source .venv/bin/activate
pip install -r requirements.txt
```

Suggested `requirements.txt`:

```text
pandas>=1.5
numpy>=1.23
scipy>=1.9
scikit-learn>=1.2
xgboost>=1.7
catboost>=1.2
shap>=0.42
matplotlib>=3.6
seaborn>=0.12
openpyxl>=3.0
matplotlib-venn>=0.11
boruta>=0.3
statsmodels>=0.13
```

If `boruta` fails to install, use the same Boruta implementation as the notebook. Chinese labels need SimSun installed locally. English figures use Times New Roman and fall back to DejaVu Serif.

\---

## Data format

Do not commit the real cohort. To rerun on synthetic data, the last column must be `Recurrence` and contain only 0/1. The other column names and order must match the final eight features. Categorical columns must be integer codes with no missing values.

```text
APTT, Depth of invasion, Pathological grade, Radiologic necrosis, Smoking, TG, Tumor location, Urine RBC count, Recurrence
```

If `DATA\_FILE` is set, the modeling cell reads this table and checks missingness, column names, and labels. Imputation, LASSO, and Boruta still have to start from the cleaned wide table. They cannot be reconstructed from these eight columns alone.

\---

## Reproduce

1. Clear notebook outputs before opening the file (see the upload checklist).
2. Point the data path at a local file that git ignores.
3. Run from top to bottom. Earlier cells create `train\_df`, `test\_df`, `best\_models`, `X\_train`, `X\_test`, and `y\_test`. SHAP, calibration, decision-curve analysis, and the DeLong test depend on those objects and cannot be run alone.
4. Figures and tables are written to `D:/两年内是否复发/` by default. Change this to `./outputs` in the public copy.

Fixed seeds: split `0`, imputation `42`, models and bootstrap `180`. On the same environment and the same data, AUC should be reproducible. Jitter in SHAP dependence plots uses a local random seed; the SHAP contributions themselves come from CatBoost `ShapValues`.

\---

## Method boundaries

The code is published so the analysis steps can be inspected. It is not a model for clinical use.

* Imaging and pathology variables are missing in about 30% of rows. Imputation does not create information from a test that was never done.
* SVM AUC is based on the decision function. Do not interpret it side by side with probability metrics from the other models.
* DeLong intervals are a normal approximation and need not match the bootstrap intervals.
* SHAP values are log-odds contributions, not hazard ratios or causal effects. Color on a categorical variable shows the code, not a continuous quantity.

\---

## Citation and license

If you use this repository, cite the peer-reviewed paper. No license is specified yet. All rights are reserved until a `LICENSE` file is added. MIT or Apache-2.0 is a common choice for research code; data and code should be licensed separately.

\---

