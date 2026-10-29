# CREDIT RISK MONITORING PROJECT

Data Used: [All Lending Club loan data](https://www.kaggle.com/datasets/wordsforthewise/lending-club) (accepted loans 2007–2018Q4)
Created by: LendingClub (published loan data; Kaggle upload by `wordsforthewise`)

## About this project
This project is a practice project to improve my skills in datascience and banking.

It builds an end-to-end **application credit scorecard with monitoring**, the way a bank model development team would:
WoE/IV scorecard (champion) → LightGBM (challenger) → calibration & decline reasons → profit-based cutoff → PSI/CSI drift monitoring → Streamlit demo and model documentation.
Public data only.

<!-- TBD (Step 7): one screenshot of the Streamlit app here -->

---

## Results

<!-- TBD (Steps 2–6): fill from reports/metrics/*.json. Never type numbers by hand. -->

| Model | Sample | AUC | Gini | KS | Brier |
|---|---|---|---|---|---|
| `sub_grade` benchmark | OOT 2015 | TBD | TBD | TBD | — |
| Scorecard (champion) | OOT 2015 | TBD | TBD | TBD | TBD |
| LightGBM (challenger) | OOT 2015 | TBD | TBD | TBD | TBD |

**Business:** cutoff at score TBD → approval rate TBD, bad rate among approved TBD, expected profit TBD per application (Step 5).
**Monitoring:** max monthly score PSI 2016–2018 = TBD; trigger status TBD (Step 6).

**Key findings** (TBD, 3 bullets max):
- TBD
- TBD
- TBD

## Data

- **Population:** TBD (Step 1), e.g. 36-month loans, …
- **Target:** TBD (Step 1), e.g. bad = Charged Off / Default; good = Fully Paid; indeterminates excluded
- **Time-based samples:** dev_train TBD · dev_valid TBD · OOT TBD · monitor TBD
- **Leakage control:** every feature is known at `issue_d`; see [`reports/column_audit.csv`](reports/column_audit.csv)

## Approach

| Step | What | Details |
|---|---|---|
| 1 | EDA, data quality, leakage checks, time-based split | [guide](docs/guide/step-01-eda-data-quality.md) |
| 2 | WoE/IV binning → logistic regression → points | [guide](docs/guide/step-02-scorecard.md) |
| 3 | LightGBM challenger, compared on AUC / Gini / KS | [guide](docs/guide/step-03-challenger.md) |
| 4 | Calibration, SHAP, plain-language decline reasons | [guide](docs/guide/step-04-calibration-explainability.md) |
| 5 | Expected-profit cutoff (approval rate vs bad rate) | [guide](docs/guide/step-05-business-cutoff.md) |
| 6 | PSI/CSI monitoring on 2016–2018 data + retrain trigger | [guide](docs/guide/step-06-monitoring.md) |
| 7 | Pipeline, tests, Streamlit app, model documentation | [guide](docs/guide/step-07-ship-it.md) |

## How to run

<!-- TBD (Step 7): confirm these commands work on a fresh clone -->

```bash
# 1. Environment (Python 3.12, uv)
uv sync --locked

# 2. Data: download accepted_2007_to_2018Q4.csv.gz from Kaggle into data/raw/lending-club/

# 3. Rebuild everything: processed data, models, metrics, monitoring reports
uv run python scripts/run_pipeline.py

# 4. Tests
uv run pytest

# 5. Demo
uv run streamlit run app/streamlit_app.py
```

## Repository structure

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md). In short: `src/credit_risk/` (logic) · `notebooks/` (analysis narrative) · `scripts/` (pipeline) · `reports/` (results) · `app/` (demo) · `docs/` (plans, guides, model documentation).

## Model documentation

[docs/MODEL_DOCUMENTATION.md](docs/MODEL_DOCUMENTATION.md) (TBD, Step 7): a 1-page summary in the style of a bank model validation submission.

## Limitations

<!-- TBD (Step 7): keep it honest and specific -->
- Accepted loans only: no reject inference, so results describe approvals *on top of* Lending Club's own policy.
- TBD
- TBD

## Project management

- Plan: [Project brief](docs/PROJECT_BRIEF.md) · [Requirements](docs/REQUIREMENTS.md) · [Roadmap](docs/ROADMAP.md)
- Progress & decisions: [docs/PROGRESS.md](docs/PROGRESS.md)
- Commit convention: [docs/COMMIT_CONVENTION.md](docs/COMMIT_CONVENTION.md)
