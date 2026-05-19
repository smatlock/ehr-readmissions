# Overview
End-to-end machine learning pipeline for predicting 30-day hospital readmissions using EHR data. Includes temporal feature engineering, leakage prevention, fairness analysis, SHAP explainability, model comparison (RF/XGBoost).

## Installation

1. Clone the repository:

```bash
git clone <repo-url>
cd ehr-readmission-risk-model
```

2. Create and activate a virtual environment:

### macOS / Linux

```bash
python3 -m venv venv
source venv/bin/activate
```

### Windows

```bash
python -m venv venv
venv\Scripts\activate
```

3. Install dependencies:

```bash
pip install -r requirements.txt
```

4. Launch Jupyter Notebook:

```bash
jupyter notebook
```

5. Open and run the notebooks in order:

* `readmissions-feature-gen.ipynb`
* `readmissions-model-dev.ipynb`

**Note, you will need to update the name of the model cohort that results from feature gen in the model dev notebook.**
