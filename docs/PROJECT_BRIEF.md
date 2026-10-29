# Project Brief: Credit Risk Scorecard with Monitoring

> **One line:** Build, challenge, explain, price-in and monitor an application credit scorecard for unsecured personal loans, using public Lending Club data, the way a bank model development team would.

---

## 1. Problem statement

A lender receives loan applications and must decide **approve or decline** for each one. If it approves too many applicants, credit losses grow. If it approves too few, it loses good customers and revenue.

The lender needs:

1. A **probability of default (PD)** for every applicant, using only information available on the day of application.
2. A **transparent score** that credit officers, auditors and regulators can read and challenge.
3. A **decision rule** (cutoff) that balances approval volume against expected losses.
4. **Monitoring** that warns when the applicant population or model behaviour drifts away from what the model was built on.

This project builds all four.

## 2. Business context

- **Product:** unsecured consumer instalment loans (Lending Club, 36-month term).
- **Why a scorecard:** points-based logistic regression scorecards are still the industry standard for application scoring. They are stable, explainable, easy to monitor, and map directly to decline reasons.
- **Why a challenger:** banks routinely benchmark the champion (the scorecard) against a more flexible challenger (gradient boosting). The challenger quantifies how much predictive power the simpler model gives up.
- **Regulatory expectations this mirrors:**
  - Model risk management, as in the US Federal Reserve/OCC **SR 11-7** guidance and similar frameworks elsewhere: documented development, out-of-time validation, ongoing monitoring.
  - Adverse action / decline reasons, as in US ECOA / Regulation B: every declined applicant must receive specific principal reasons.
- **Why monitoring:** a model is built on a past population. Marketing campaigns, economic cycles and policy changes all shift who applies. Monitoring with PSI/CSI is how a bank knows when to recalibrate or rebuild.

## 3. Dataset

| Item | Value |
|---|---|
| Source | Kaggle, [All Lending Club loan data](https://www.kaggle.com/datasets/wordsforthewise/lending-club) (published by LendingClub, uploaded by `wordsforthewise`) |
| File used | `accepted_2007_to_2018Q4.csv.gz` (about 392 MB compressed, about 2.26M loans, 151 columns) |
| Data dictionary | `LCDataDictionary.xlsx` (from Kaggle [Lending Club Loan Data](https://www.kaggle.com/datasets/wendykan/lending-club-loan-data) or the [figshare copy](https://figshare.com/articles/dataset/Lending_club_dataset_description/20016077)) |
| Time field | `issue_d`: month of loan origination. Used for the out-of-time split and for monitoring cohorts |
| Why this dataset | It is the only candidate (vs. Home Credit, Give Me Some Credit) with a real origination date, which enables an honest out-of-time test and real drift |

**Hard constraint:** public data only. No internal bank data is used, referenced or imitated anywhere in this repo.

## 4. Scope

| # | Step | In scope |
|---|---|---|
| 1 | EDA & data quality | Column audit, target and population definition, vintage analysis, missingness, leakage checks, time-based split |
| 2 | Classic scorecard | WoE/IV binning, variable selection, logistic regression, points scaling, AUC/Gini/KS |
| 3 | Challenger | LightGBM on the same population, compared on AUC, Gini, KS (with uncertainty) |
| 4 | Calibration & explainability | Reliability curves, Brier score, Platt/isotonic calibration, SHAP, plain-language decline reasons |
| 5 | Business layer | Expected-profit model, strategy curve (approval rate vs bad rate), cutoff choice, sensitivity |
| 6 | Monitoring | Score PSI, characteristic CSI, monthly reports on 2016–2018 data, synthetic shock test, retrain trigger |
| 7 | Ship it | One-command pipeline, tests, Streamlit demo, README, 1-page model documentation |

## 5. Non-goals

These are deliberately **out of scope**. Mention them as limitations, but do not build them:

- Reject inference. The data contains only loans Lending Club approved.
- Pricing, limit setting or loan amount optimisation.
- Regulatory capital (Basel IRB) or IFRS 9 / CECL lifetime expected loss models.
- A full fair-lending audit (but geographic fields are excluded or justified; see Step 1).
- Production infrastructure: real-time serving, authentication, databases, CI/CD to a cloud provider.
- Deep learning or extensive AutoML.

## 6. Success criteria

| Area | Criterion |
|---|---|
| Data integrity | A written column audit shows every model feature is known at `issue_d`. An automated test fails if a post-origination column enters a feature list |
| Validation design | All headline metrics are reported on an **out-of-time** sample that was never used for fitting, tuning, binning or calibration |
| Discrimination | Scorecard and challenger AUC/Gini/KS are reported on dev-train, dev-valid and OOT. The OOT Gini drop vs dev-train is explained. **Sanity band:** for application-only Lending Club features, OOT AUC is typically around 0.65–0.72. A result far above that is treated as a leakage alarm, not a success |
| Benchmark | Scorecard Gini is compared with Lending Club's own `sub_grade` used as a single-variable ranking |
| Calibration | OOT reliability diagram, Brier score, and mean predicted PD vs observed bad rate are reported before and after calibration |
| Explainability | Every declined applicant gets up to 4 plain-language reasons. Three worked examples are documented |
| Business | Cutoff chosen from an expected-profit curve with stated assumptions, and a sensitivity analysis showing how the cutoff moves |
| Monitoring | A generated monthly report (score PSI + CSI per characteristic, RAG status) for 2016–2018, plus a synthetic shock that correctly fires the retrain trigger |
| Reproducibility | On a fresh clone with the raw file in place, `uv sync --locked` followed by one pipeline command regenerates the models and metrics (matching to 3 decimals) |
| Communication | README, 1-page model documentation and a working Streamlit demo |

## 7. Final deliverables

1. `src/credit_risk/`: reusable, tested Python package (data, binning, scorecard, challenger, calibration, explain, business, monitoring, metrics).
2. `notebooks/01–07`: narrative notebooks, one per stage, each runs top to bottom.
3. `scripts/run_pipeline.py`: one command that rebuilds everything from raw data.
4. `reports/`: column audit, figures, metrics JSON, scorecard table, monitoring reports.
5. `app/streamlit_app.py`: scoring demo with decision, PD and decline reasons, plus a monitoring view.
6. `docs/MODEL_DOCUMENTATION.md`: 1-page model document, filled in from the template.
7. `README.md`: portfolio-quality project front page.

## 8. Key decisions to finalise in Step 1

These are drafts. Confirm or change them, with reasons, in `docs/PROGRESS.md`:

- **Population:** 36-month loans. Treat "Does not meet the credit policy" loans and joint applications as to be decided.
- **Target:** bad = `Charged Off` / `Default` (plus their policy variants); good = `Fully Paid`; everything still running = indeterminate (excluded).
- **Windows:** development = issued up to 2014-12 (last months held out as dev-valid); OOT = 2015; monitoring = 2016-01 to 2018-12.
- **LC risk outputs** (`grade`, `sub_grade`, `int_rate`, `installment`): used as features or excluded?
