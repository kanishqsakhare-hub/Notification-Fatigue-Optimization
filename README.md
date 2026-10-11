qefv;/ker\r=\   ==ppp00-9-01290-io/cvheddnkl4343# Notification Fatigue Optimization

This project studies a common notification problem: sending more messages creates more chances for engagement, but it can also increase fatigue and opt-outs. The analysis looks at when a notification should be sent immediately, delayed, or suppressed.

The repository uses synthetic data for 5,000 users and 85,019 notification opportunities between January and March 2026. The model results and policy comparisons describe this dataset and its simulated behavior. They are not production measurements or causal estimates.

## What the project does

The work is organized as five notebooks. Together they validate the source data, explore notification behavior, train click and engagement models, define a fatigue-aware policy, and evaluate that policy in a user-randomized simulation.

The main questions are:

- How are notification frequency and fatigue related to engagement?
- Can candidate-time features predict clicks or meaningful engagement in a later period?
- What happens to engagement and notification volume when high-fatigue or low-value sends are delayed or suppressed?
- Does the policy continue to respect consent, delivery limits, and delayed-message expiry as user state changes?

The notebooks are self-contained. Shared validation, modeling, simulation, and statistical functions are defined inside the notebooks rather than imported from local project modules.

## How it works

Notebook 01 checks event keys, timestamps, user joins, missing values, response denominators, and historical exposure counts. It also handles an invalid churn field conservatively. The available history supports 43,022 future-open negatives, but it does not support positive churn labels or population churn modeling.

Notebook 02 describes delivery, click, engagement, fatigue, opt-out, and segment behavior. Association tests are performed at user level so users with more notification events do not automatically receive more weight.

Notebook 03 trains logistic-regression models for `clicked_24h` and `engaged_24h`. Categorical columns are one-hot encoded and numeric columns are standardized with a scikit-learn `ColumnTransformer`. Training uses events before March 1 whose 24-hour response windows are complete. March events with complete windows form the future holdout. A prevalence-only `DummyClassifier` provides the baseline.

Notebook 04 defines the policy. Thresholds are estimated from the training period: fatigue above `0.1530` can trigger a delay, fatigue above `0.1926` can trigger suppression when the engagement score is low, and prior-week exposure of `2` is treated as high. The simulator processes each user's events chronologically. It updates fatigue and exposure after actual simulated deliveries, retains opt-out state, retries delayed notifications at 09:00, and expires unresolved messages after 72 hours.

Notebook 05 assigns 2,283 users to each arm and runs the policy as a synthetic A/B test. Effects are calculated from user-level outcomes with 2,000 bootstrap samples. Users who receive no notifications remain in assigned-user metrics.

## Tech stack

- Python and JupyterLab
- pandas and NumPy
- scikit-learn
- SciPy
- Matplotlib and Seaborn
- nbformat and nbconvert

Exact versions are pinned in `requirements.txt`.

## Repository structure

```text
.
├── README.md
├── DATA_DICTIONARY.md
├── requirements.txt
├── data/
│   ├── raw/
│   │   ├── users.csv
│   │   └── notification_events.csv
│   └── processed/
│       └── notification_features.csv
└── notebooks/
    ├── 01_data_understanding_validation.ipynb
    ├── 02_exploratory_data_analysis.ipynb
    ├── 03_predictive_modeling.ipynb
    ├── 04_notification_policy_optimization.ipynb
    └── 05_ab_testing_business_impact.ipynb
```

`notification_features.csv` is a required input. Other files created under `data/processed/` are reproducible notebook outputs and are ignored by Git.

## Setup and execution

Create a virtual environment and install the pinned dependencies:

```bash
python -m venv .venv
```

Windows PowerShell:

```powershell
.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
jupyter lab
```

macOS or Linux:

```bash
source .venv/bin/activate
python -m pip install -r requirements.txt
jupyter lab
```

Start Jupyter from the repository root. Run each notebook from a fresh kernel in this order:

1. `01_data_understanding_validation.ipynb`
2. `02_exploratory_data_analysis.ipynb`
3. `03_predictive_modeling.ipynb`
4. `04_notification_policy_optimization.ipynb`
5. `05_ab_testing_business_impact.ipynb`

## Results and observations

Among delivered events, the click rate is `32.30%`, the engagement rate is `36.20%`, and the opt-out rate is `0.187%`. Prior-week notification frequency has a small positive association with user engagement (`ρ = 0.0667`), while pre-send fatigue has a negative association (`ρ = -0.1410`). These are associations, not treatment effects.

The future-holdout model results are:

| Target and model | ROC AUC | Average precision | Brier loss |
|---|---:|---:|---:|
| Click baseline | 0.5000 | 0.3267 | 0.2200 |
| Click logistic regression | 0.6188 | 0.4257 | 0.2116 |
| Engagement baseline | 0.5000 | 0.3652 | 0.2318 |
| Engagement logistic regression | 0.6062 | 0.4556 | 0.2244 |

The user-cluster bootstrap 95% ROC AUC intervals are `[0.6102, 0.6281]` for click and `[0.5977, 0.6155]` for engagement. The opt-out model is much less reliable: its ROC AUC is `0.5617` with a 95% interval of `[0.4587, 0.6623]`, based on only 29 positive holdout events.

In the randomized policy simulation:

| Metric | Control | Fatigue-aware | Difference (95% bootstrap CI) |
|---|---:|---:|---:|
| Engagements / delivered notification | 0.3609 | 0.3672 | +0.0063 [-0.0076, 0.0206] |
| Engagements / assigned user | 1.5777 | 1.4258 | -0.1520 [-0.2203, -0.0832] |
| Notifications / assigned user | 4.3719 | 3.8826 | -0.4893 [-0.5826, -0.3938] |
| Opt-outs / assigned user | 0.0088 | 0.0061 | -0.0026 [-0.0074, 0.0026] |

The policy sends fewer notifications and produces fewer engagements per assigned user. Its estimated efficiency and opt-out differences both include zero, so this simulation does not show a clear rollout benefit.

## Limitations

The data and response process are synthetic. The simulator reuses a fixed historical opportunity stream and cannot model changes in organic activity or future notification demand. Some fatigue and propensity fields also have incomplete generation-time lineage.

Complete daily activity history is unavailable, so churn and retention cannot be validated. Opt-outs are rare, which makes calibration and safety comparisons uncertain. The source also lacks ingestion timestamps and timezone metadata, so response maturity is based on calendar boundaries rather than observed data-arrival times.

A real evaluation would require a prospective user-randomized experiment, complete consent and activity histories, prespecified guardrails, and enough opt-out events to assess safety with useful precision.
