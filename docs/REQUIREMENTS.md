# Requirements

**MUST** = required for the step to be done. **SHOULD** = expected unless you document why not. **COULD** = stretch.
IDs (e.g. `FR-2.4`) let you reference requirements in commits, PROGRESS.md and the model documentation.

---

## 1. Functional requirements

### Step 1: EDA, data quality, leakage, split

| ID | Level | Requirement |
|---|---|---|
| FR-1.1 | MUST | Load the accepted-loans file with explicit column selection and dtypes; persist a typed interim parquet so the 392 MB CSV is read once |
| FR-1.2 | MUST | Produce `reports/column_audit.csv` classifying **all 151 columns** as one of: `application_time`, `post_origination`, `lc_risk_output`, `id_admin`, `text_high_cardinality`, `target_related`, each with keep/drop and a reason |
| FR-1.3 | MUST | Define population and target in code; produce a filter "waterfall" (row count after each filter) |
| FR-1.4 | MUST | Vintage analysis: loan volume and bad rate by issue month (by term); discuss right-censoring |
| FR-1.5 | MUST | Missingness report: overall % per kept column **and** by issue year (detects fields introduced mid-history) |
| FR-1.6 | MUST | Time-based split into `dev_train`, `dev_valid`, `oot`, `monitor`; saved as parquet; no ID overlap (asserted) |
| FR-1.7 | SHOULD | Single-feature AUC screen on dev-train to flag suspiciously strong features |
| FR-1.8 | SHOULD | Short data quality summary (findings + treatments) at the end of the EDA notebook |

### Step 2: Classic scorecard

| ID | Level | Requirement |
|---|---|---|
| FR-2.1 | MUST | Fine/coarse binning fit on `dev_train` only; missing values in their own bin; minimum bin size documented |
| FR-2.2 | MUST | WoE and IV per variable; IV table saved to `reports/` |
| FR-2.3 | MUST | Variable selection using IV, correlation between WoE variables, coefficient sign/significance and business sense; each drop has a reason |
| FR-2.4 | MUST | Logistic regression on WoE features; coefficient signs consistent with the WoE convention |
| FR-2.5 | MUST | Points scaling with documented PDO, base score and base odds; scorecard table (variable, bin, WoE, points) saved |
| FR-2.6 | MUST | AUC, Gini, KS on dev-train, dev-valid, OOT; gains table by score band |
| FR-2.7 | SHOULD | Benchmark against `sub_grade` as a single-variable ranker |
| FR-2.8 | COULD | Hand-implemented WoE/IV reconciled against the library result |

### Step 3: Challenger

| ID | Level | Requirement |
|---|---|---|
| FR-3.1 | MUST | LightGBM trained on the **same rows and candidate features** as the scorecard (raw values, not WoE) |
| FR-3.2 | MUST | Early stopping and tuning use `dev_valid` only; fixed seed; deterministic training |
| FR-3.3 | MUST | Comparison table: scorecard vs challenger on AUC, Gini, KS for all splits, same metric functions |
| FR-3.4 | SHOULD | Bootstrap confidence interval for the OOT Gini difference |
| FR-3.5 | COULD | Monotone constraints on key features; report the Gini cost |

### Step 4: Calibration & explainability

| ID | Level | Requirement |
|---|---|---|
| FR-4.1 | MUST | Reliability diagram, Brier score, mean predicted PD vs observed bad rate, for both models on dev-valid and OOT |
| FR-4.2 | MUST | Calibrator (Platt and/or isotonic) fit on `dev_valid`; evaluated on OOT; AUC unchanged (asserted) |
| FR-4.3 | MUST | SHAP global importance and at least 3 local explanations for LightGBM |
| FR-4.4 | MUST | Decline-reason function returning up to 4 plain-language reasons per applicant, for the scorecard (points-below-max method) |
| FR-4.5 | SHOULD | Same for LightGBM via SHAP; compare reason agreement between models on a sample |
| FR-4.6 | MUST | Three worked examples (clear approve, clear decline, borderline) written up |

### Step 5: Business layer

| ID | Level | Requirement |
|---|---|---|
| FR-5.1 | MUST | Written profit assumptions (revenue if good, loss if bad: EAD, LGD, funding cost) |
| FR-5.2 | MUST | Strategy table across cutoffs: approval rate, bad rate among approved, expected profit |
| FR-5.3 | MUST | Chosen cutoff with justification (profit-max and/or risk-appetite constraint); saved for the app |
| FR-5.4 | MUST | Sensitivity of the cutoff to at least LGD and revenue assumptions |
| FR-5.5 | COULD | Realised-profit back-test using actual repayments (`total_pymnt`, `recoveries`). These are allowed as *outcomes*, never as features |
| FR-5.6 | COULD | Swap-set analysis: scorecard vs challenger at equal approval rate |

### Step 6: Monitoring

| ID | Level | Requirement |
|---|---|---|
| FR-6.1 | MUST | `psi()` and `csi()` functions with fixed development bin edges and empty-bin handling; unit-tested |
| FR-6.2 | MUST | Monthly score PSI and per-characteristic CSI for 2015-01 to 2018-12 vs development |
| FR-6.3 | MUST | Trigger rules (thresholds, persistence, actions) defined in config, not hard-coded in notebooks |
| FR-6.4 | MUST | Synthetic shock scenario that fires the trigger; documented as synthetic |
| FR-6.5 | MUST | Script generates a monitoring report (markdown or HTML) for a given month with RAG status |
| FR-6.6 | SHOULD | Performance monitoring (Gini, calibration) by vintage where outcomes are mature, with censoring caveat |

### Step 7: Ship it

| ID | Level | Requirement |
|---|---|---|
| FR-7.1 | MUST | `scripts/run_pipeline.py` rebuilds processed data, models, metrics and monitoring reports from raw data |
| FR-7.2 | MUST | `uv run pytest` passes (metrics, WoE/IV, scaling, PSI, leakage guard) |
| FR-7.3 | MUST | Streamlit app: score an applicant → score, PD, decision, reasons; performance and monitoring views |
| FR-7.4 | MUST | App runs from a small committed public sample, not the raw file |
| FR-7.5 | MUST | `docs/MODEL_DOCUMENTATION.md` (≤ 1 page) and complete README |
| FR-7.6 | COULD | Deployed on Streamlit Community Cloud; GitHub Actions running `ruff` + `pytest` |

---

## 2. Technical requirements

| ID | Requirement |
|---|---|
| TR-1 | Python 3.12, managed by **uv**. `pyproject.toml`, `uv.lock` and `.python-version` are committed; `uv sync --locked` recreates the exact environment |
| TR-2 | Reusable code lives in the `src/credit_risk/` package (src layout), importable from notebooks, scripts, tests and the app |
| TR-3 | One configuration module, `src/credit_risk/config.py`, holds paths (built with `pathlib` from the project root), `SEED`, date windows, feature lists and thresholds. No absolute paths, no `os.getcwd()`-relative paths |
| TR-4 | Randomness is seeded everywhere: numpy `Generator`, scikit-learn `random_state`, LightGBM `seed` + `deterministic=True` (+ `force_row_wise` or `force_col_wise`) |
| TR-5 | Cross-platform: runs on Windows (your machine) and Linux (CI / Streamlit Cloud) |
| TR-6 | Tests with `pytest`; linting and formatting with `ruff` |
| TR-7 | Notebooks pass "Restart Kernel & Run All" without errors; first cell states the purpose, last cell states conclusions |
| TR-8 | Model artifacts saved with `joblib`/JSON alongside metadata: training window, feature list, library versions, git commit hash |
| TR-9 | Laptop-friendly: load with `usecols`/dtypes, work from parquet; full pipeline SHOULD finish in under ~30 minutes |
| TR-10 | pandas 3.x semantics: Copy-on-Write is the default, so chained assignment (`df["a"][mask] = x`) does not modify `df`. Use `.loc` |

## 3. Data requirements

| ID | Requirement |
|---|---|
| DR-1 | Source: Kaggle `wordsforthewise/lending-club`, file `accepted_2007_to_2018Q4.csv.gz` only. Record download date, row count and SHA-256 in `docs/PROGRESS.md` |
| DR-2 | **Public data only.** No internal bank data, real customer data, or data "inspired by" work systems, anywhere: notebooks, tests, app samples, screenshots |
| DR-3 | `data/raw/` is immutable. Every derived dataset is reproducible from raw via code |
| DR-4 | `data/` and `models/` are gitignored. Only small public samples (e.g. ≤ 500 rows for the app) are committed |
| DR-5 | Observation point = `issue_d`. A feature is allowed only if its value is known at `issue_d` |
| DR-6 | Target, population filters and exclusions are documented with row counts (waterfall) |
| DR-7 | Windows (draft, finalise in Step 1): dev ≤ 2014-12 (with the last months as dev-valid), OOT = 2015, monitor = 2016-01 to 2018-12 |
| DR-8 | Fair lending: geographic fields (`zip_code`, `addr_state`) and free text (`emp_title`, `desc`, `title`) are excluded by default. Any use must be justified in the decision log |
| DR-9 | Monitoring-period outcomes are never used for training or for choosing thresholds |

## 4. Definition of done

### Per step

- [ ] All MUST requirements for the step are met, and the guide's acceptance criteria pass
- [ ] Notebook(s) pass Restart & Run All
- [ ] Logic needed by later steps is moved into `src/credit_risk/`, with tests where the guide asks
- [ ] `docs/PROGRESS.md` updated: status, hours, metrics, decisions, answers to the self-check questions
- [ ] Work committed using [docs/COMMIT_CONVENTION.md](COMMIT_CONVENTION.md)

### Whole project

- [ ] Fresh clone + raw file → `uv sync --locked` → `uv run python scripts/run_pipeline.py` reproduces the metrics in PROGRESS.md (3 decimals)
- [ ] `uv run pytest` green
- [ ] `uv run streamlit run app/streamlit_app.py` works
- [ ] README and model documentation complete; numbers match `reports/metrics/`
- [ ] Quality checklist below fully ticked
- [ ] `dev` merged into `main` via PR and tagged `v1.0.0`

## 5. Quality checklist

### Leakage

- [ ] Every model feature is `application_time` in `reports/column_audit.csv`
- [ ] A test fails if any `post_origination` column appears in a feature list
- [ ] Binning, imputation, capping and encoders are fit on `dev_train` only
- [ ] Early stopping, tuning and calibration use `dev_valid` only; OOT is touched only for final evaluation
- [ ] Any feature with single-feature AUC > ~0.80 or IV > 0.5 has been investigated and the outcome written down
- [ ] Decision on `grade` / `sub_grade` / `int_rate` / `installment` documented
- [ ] No monitoring-period labels used in training or threshold setting

### Reproducibility

- [ ] `uv.lock` committed; `uv sync --locked` succeeds on a clean checkout
- [ ] `SEED` comes from config and is used by every random operation
- [ ] All paths come from config (no `C:\...`, no backslash literals)
- [ ] Raw file SHA-256 recorded and checked by the pipeline
- [ ] Pipeline reruns produce identical metrics
- [ ] Notebooks committed only after Restart & Run All

### Documentation

- [ ] Decision log covers: population, target, windows, exclusions, feature selection, scaling parameters, calibration method, cutoff, monitoring thresholds
- [ ] Every notebook has a purpose and a conclusion cell
- [ ] Model documentation fits on 1 page and its numbers match the reports
- [ ] Limitations stated: accepted-only bias, censoring, 36-month-only scope, profit assumptions, synthetic shock
- [ ] README explains how to run everything in ≤ 5 commands
