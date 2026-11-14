# Assignment 4: Reweighing (Kamiran & Calders, 2012) on Adult, German and COMPAS

This folder is part of my fork of [Trusted-AI/AIF360](https://github.com/Trusted-AI/AIF360). The AIF360 library itself is used unchanged (version 0.6.1, `aif360.algorithms.preprocessing.Reweighing`). Everything specific to the assignment is in `assignment4/`.

| File | Purpose |
|---|---|
| `reweighing_evaluation.ipynb` | Full pipeline for Task 2, saved **with outputs** |
| `requirements.txt` | Exact package versions of the environment I used |
| `results/` | CSV tables and figure written by the notebook |

## What is evaluated

- **Datasets and protected attribute:** Adult (`sex`), German Credit (`sex`), COMPAS (`race`), loaded with the default AIF360 loaders and default pre-processing (Adult: 45,222 rows after dropping rows with missing values; German: 1,000; COMPAS: 6,167).
- **Privileged group** is value 1 of the protected attribute in AIF360 (Adult: Male, German: male, COMPAS: Caucasian).
- **Favourable label:** 1 for Adult (income > 50K) and German (good credit). For COMPAS the stored label is two-year recidivism (1 = reoffended) and the favourable label is 0 (no reoffending). F1 for COMPAS therefore refers to the label 1 class.
- **Measures:**
  - `DP_Diff` = P(favourable | unprivileged) - P(favourable | privileged). 0 is fair, negative means the unprivileged group is disadvantaged. This is AIF360's sign; the paper uses the opposite one.
  - `DIR` = P(favourable | unprivileged) / P(favourable | privileged). 1 is fair.
  - `Consistency` with k = 5 nearest neighbours on standardised features.
  - Accuracy, balanced accuracy and F1 as predictive-performance measures.
- **Before mitigation:** measures on the original labels of each full dataset (notebook Section 3).
- **After mitigation:** measures on the predicted labels of the held-out test split, for a baseline logistic regression and for logistic regression trained with the Reweighing weights (Sections 4 and 5).

## Parameter settings

| Parameter | Value |
|---|---|
| Train/test split | 70 / 30, stratified on (label, protected attribute) |
| Seeds | 0 to 9 (10 splits, results are mean ± std) |
| Feature scaling | `StandardScaler`, fitted on the training part only |
| Classifier | scikit-learn `LogisticRegression(max_iter=1000)`, other settings default |
| Reweighing | AIF360 defaults, privileged = `{attr: 1}`, unprivileged = `{attr: 0}` |
| Consistency | k = 5, own brute-force helper, checked against AIF360's implementation on German (difference 0.00000, notebook Section 2b) |

## How to reproduce

Tested on macOS with Python 3.14.4, numpy 2.5.4, pandas 2.3.3, scikit-learn 1.9.1 and aif360 0.6.1. On Windows, activate the environment with `.venv\Scripts\activate` instead of `source .venv/bin/activate`.

1. **Get the code** and go to this folder:
   ```bash
   git clone https://github.com/laviniabondoni/AIF360.git
   cd AIF360/assignment4
   ```
2. **Create an environment and install the packages** (option A reproduces my exact versions):
   ```bash
   python3 -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
   ```
   Option B, if option A fails on your Python version:
   ```bash
   pip install aif360 scikit-learn "pandas<3" matplotlib ipykernel jupyter
   ```
   **pandas must be below 3.** With pandas 3, the Adult loader of AIF360 stops with `TypeError: Cannot interpret '<StringDtype ...>' as a data type`.
3. **Open the notebook** `reweighing_evaluation.ipynb` in VS Code (extensions Python and Jupyter; choose the `.venv` environment as kernel) or run `jupyter notebook reweighing_evaluation.ipynb`.
4. Click **Restart**, then **Run All**. In my run Adult took about 45 s, COMPAS about 25 s and German under 1 s.
5. Results appear in the notebook and are written to `results/` (next to the notebook): `original_labels.csv`, `results_per_seed.csv`, `results_summary.csv`, `paired_change.csv`, `comparison.png`.

### The datasets do not download (common on macOS)

The notebook tries to download the raw data into AIF360's `data/raw` folder. If it prints `COULD NOT DOWNLOAD` or the Adult cell stops with an `IOError: No such file or directory`, download the files with `curl` instead (inside the activated environment):

```bash
cd "$(python -c "import aif360,os;print(os.path.join(os.path.dirname(aif360.__file__),'data','raw'))")"
mkdir -p adult german compas
curl -L -o adult/adult.data https://archive.ics.uci.edu/ml/machine-learning-databases/adult/adult.data
curl -L -o adult/adult.test https://archive.ics.uci.edu/ml/machine-learning-databases/adult/adult.test
curl -L -o adult/adult.names https://archive.ics.uci.edu/ml/machine-learning-databases/adult/adult.names
curl -L -o german/german.data https://archive.ics.uci.edu/ml/machine-learning-databases/statlog/german/german.data
curl -L -o german/german.doc https://archive.ics.uci.edu/ml/machine-learning-databases/statlog/german/german.doc
curl -L -o compas/compas-scores-two-years.csv https://raw.githubusercontent.com/propublica/compas-analysis/master/compas-scores-two-years.csv
cd -
```

Expected sizes: `adult.data` about 3.8 MB, `adult.test` about 1.9 MB, `compas-scores-two-years.csv` about 2.4 MB, `german.data` about 78 KB. Then restart the notebook and run all cells again. Do not commit the raw data files.

## Results of the saved run (mean ± std over 10 splits, held-out test data)

| Dataset | Measure | Original labels | No mitigation | Reweighing |
|---|---|---|---|---|
| Adult | DP_Diff | -0.199 | -0.186 ± 0.005 | -0.087 ± 0.006 |
| | DIR | 0.363 | 0.292 ± 0.014 | 0.599 ± 0.019 |
| | Accuracy | - | 0.848 ± 0.003 | 0.842 ± 0.003 |
| German | DP_Diff | -0.075 | -0.134 ± 0.073 | -0.072 ± 0.077 |
| | DIR | 0.897 | 0.837 ± 0.086 | 0.911 ± 0.095 |
| | Accuracy | - | 0.742 ± 0.016 | 0.737 ± 0.020 |
| COMPAS | DP_Diff | -0.097 | -0.165 ± 0.031 | -0.006 ± 0.023 |
| | DIR | 0.840 | 0.770 ± 0.037 | 0.990 ± 0.036 |
| | Accuracy | - | 0.668 ± 0.007 | 0.666 ± 0.009 |

Consistency, balanced accuracy and F1 are in the notebook output and in `results/results_summary.csv`.

## Notes

- Warnings about `tensorflow`, `fairlearn` and `inFairness` at the start are harmless: they only concern AIF360 algorithms that this assignment does not use.
- The protected attribute stays in the feature matrix (AIF360 default). Reweighing changes only the training weights, and the test data is never modified.
- German has only 1,000 rows, so single splits are noisy; this is why all numbers are averaged over 10 splits.
