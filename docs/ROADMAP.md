# Roadmap

## Time budget

You said **6–10 hours per week (uncertain)**. The plan below sizes the **core** work at about **33.5 h**, plus about 14.5 h of optional stretch work.

| Your actual pace | What happens |
|---|---|
| ~6–7 h/week | 5 weeks: Weeks 1–4 as below, Step 7 in Week 5 |
| ~9–10 h/week | 4 weeks: pull Step 7 into Week 4 and use the spare time on stretch tasks |
| < 6 h/week | Keep the order, let the dates slide, and cut stretch tasks first |

**If you fall behind:** drop stretch tasks first. **Never drop** the leakage checks, the out-of-time evaluation, the decision log, or the model documentation. Those are what make this a credit risk project rather than a Kaggle notebook.

## Timeline

| Week | Dates (2026) | Steps | Core h | Milestone |
|---|---|---|---|---|
| 0 | Thu 29 Oct – Sun 1 Nov | Step 0: setup | 1.5 | **M0:** environment locked, data downloaded, folders and first commits in place |
| 1 | Mon 2 – Sun 8 Nov | Step 1: EDA & data quality | 6 | **M1:** column audit, target, split parquet files, DQ summary |
| 2 | Mon 9 – Sun 15 Nov | Step 2: scorecard | 7 | **M2:** scorecard table + AUC/Gini/KS on OOT |
| 3 | Mon 16 – Sun 22 Nov | Step 3: challenger · Step 4: calibration & explainability | 3 + 4 | **M3:** champion vs challenger table, calibrated PD, decline reasons |
| 4 | Mon 23 – Sun 29 Nov | Step 5: business · Step 6: monitoring | 3 + 4 | **M4:** cutoff chosen, monthly monitoring report + trigger |
| 5 | Mon 30 Nov – Sun 6 Dec | Step 7: ship it (+ buffer) | 5 | **M5 / v1.0.0:** pipeline, tests, app, README, model doc |

## Hours per step

| Step | Core | Stretch | Guide |
|---|---|---|---|
| 0 Setup | 1.5 | — | [step-00](guide/step-00-setup.md) |
| 1 EDA & data quality | 6 | 2 | [step-01](guide/step-01-eda-data-quality.md) |
| 2 Classic scorecard | 7 | 2 | [step-02](guide/step-02-scorecard.md) |
| 3 Challenger (LightGBM) | 3 | 2 | [step-03](guide/step-03-challenger.md) |
| 4 Calibration & explainability | 4 | 2 | [step-04](guide/step-04-calibration-explainability.md) |
| 5 Business layer | 3 | 1.5 | [step-05](guide/step-05-business-cutoff.md) |
| 6 Monitoring | 4 | 2 | [step-06](guide/step-06-monitoring.md) |
| 7 Ship it | 5 | 3 | [step-07](guide/step-07-ship-it.md) |
| **Total** | **33.5** | **14.5** | |

---

## Task list

Tick the boxes here or in [PROGRESS.md](PROGRESS.md). Your choice, but keep one place as the source of truth.

### Week 0: Step 0 Setup (M0)

- [ ] Download `accepted_2007_to_2018Q4.csv.gz` + data dictionary into `data/raw/lending-club/`; record SHA-256
- [ ] `uv init --lib`, add dependencies, commit `pyproject.toml` + `uv.lock`
- [ ] Extend `.gitignore`
- [ ] Create `config.py` with project-root-based paths and `SEED`
- [ ] VS Code kernel points at `.venv`; import check passes
- [ ] First conventional commits pushed to `dev`

### Week 1: Step 1 EDA & data quality (M1)

- [ ] Rewrite `01_extract_data.ipynb` for Lending Club → `data/interim/loans.parquet`
- [ ] Column audit for all 151 columns → `reports/column_audit.csv`
- [ ] Target + population definition, filter waterfall
- [ ] Vintage analysis (volume & bad rate by issue month)
- [ ] Missingness overall and by year
- [ ] Univariate checks + treatment plan (outliers, invalid values)
- [ ] Single-feature AUC leakage screen
- [ ] Time-based split → `data/processed/*.parquet`, no ID overlap
- [ ] Promote loading/target/split code to `src/credit_risk/data.py`
- [ ] DQ summary + decision log entries
- [ ] *Stretch:* vintage curves by grade; monthly volume vs bad-rate dual-axis chart for README

### Week 2: Step 2 Scorecard (M2)

- [ ] Hand-compute WoE/IV for 1 numeric + 1 categorical feature
- [ ] Binning on dev_train (monotonic where sensible, min bin size, missing bin)
- [ ] IV table, drop weak, investigate suspicious
- [ ] Correlation/VIF check between WoE features
- [ ] Logistic regression on WoE; signs & significance; iterate to 8–15 features
- [ ] Points scaling (PDO / base score / base odds) → scorecard table
- [ ] AUC / Gini / KS + gains table on all splits; `sub_grade` benchmark
- [ ] `metrics.py` + tests
- [ ] *Stretch:* manual bin adjustments with business rationale; CSI preview dev_train vs OOT

### Week 3: Step 3 Challenger + Step 4 Calibration & explainability (M3)

- [ ] LightGBM baseline on same rows; early stopping on dev_valid; deterministic
- [ ] Small tuning run (≤ 20 trials) on dev_valid
- [ ] Comparison table champion vs challenger
- [ ] *Stretch:* bootstrap CI on Gini difference; monotone constraints
- [ ] Reliability diagrams + Brier + calibration-in-the-large
- [ ] Fit calibrator on dev_valid; evaluate on OOT; assert AUC unchanged
- [ ] SHAP global + 3 local explanations
- [ ] Reason codes for scorecard (points below max) + plain-language mapping
- [ ] Three worked decline/approve examples
- [ ] *Stretch:* SHAP-based reasons for LightGBM; agreement analysis

### Week 4: Step 5 Business + Step 6 Monitoring (M4)

- [ ] Write profit assumptions
- [ ] Strategy table + profit curve; choose cutoff → `models/cutoff.json`
- [ ] Sensitivity analysis (LGD, revenue)
- [ ] *Stretch:* realised-profit back-test; swap-set analysis
- [ ] `psi()` / `csi()` + tests
- [ ] Monthly score PSI + CSI heatmap 2015–2018
- [ ] Trigger rules in config
- [ ] Synthetic shock fires trigger
- [ ] `scripts/monitor.py --month` writes a report
- [ ] *Stretch:* vintage performance monitoring with censoring caveat

### Week 5: Step 7 Ship it (M5 → v1.0.0)

- [ ] `scripts/run_pipeline.py` end to end; rerun gives identical metrics
- [ ] Full test suite green
- [ ] Streamlit app (score / performance / monitoring)
- [ ] App sample file (≤ 500 public rows)
- [ ] `docs/MODEL_DOCUMENTATION.md` (1 page)
- [ ] README complete with results and screenshots
- [ ] Quality checklist in REQUIREMENTS.md all ticked
- [ ] PR `dev` → `main`, tag `v1.0.0`
- [ ] *Stretch:* Streamlit Community Cloud deployment; GitHub Actions (ruff + pytest)
