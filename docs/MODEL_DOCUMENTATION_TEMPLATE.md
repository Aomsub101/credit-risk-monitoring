<!--
HOW TO USE THIS TEMPLATE
- Copy to docs/MODEL_DOCUMENTATION.md in Step 7 and replace every [bracket].
- Hard limit: one printed page (~600–700 words). Validators read many of these; brevity is respect.
- Every number must come from reports/metrics/ — cite the file, never retype from memory.
- Write in plain declarative sentences. State facts, then limitations. No marketing language.
- Delete this comment block in the final version.
-->

# Model Documentation: [Model name, e.g. "LC Personal Loan Application Scorecard"]

| Model ID | [e.g. LC-APP-PD-001] | Version | [v1.0.0 / git tag] | Date | [YYYY-MM-DD] |
|---|---|---|---|---|---|
| **Owner / developer** | [Your name] | **Model type** | [Logistic regression scorecard (champion); LightGBM (challenger, benchmark only)] | **Status** | [Development — practice project, public data] |

## 1. Purpose and intended use

[One or two sentences: what decision the model supports and for which product/population. E.g. "Estimates 36-month probability of charge-off at application for unsecured personal loans, to support approve/decline decisions at a single score cutoff."]
**Not intended for:** [e.g. pricing, limit setting, capital calculation, populations outside 36-month unsecured loans.]

## 2. Data

- **Source:** [Lending Club accepted loans 2007–2018Q4, Kaggle `wordsforthewise/lending-club`; SHA-256 `[…]`.]
- **Population:** [filters applied; reference the waterfall in PROGRESS.md.]
- **Target:** [bad definition; good definition; indeterminates excluded and their share.]
- **Samples:** [dev_train / dev_valid / OOT / monitor: periods, N, bad rate — one compact line each or a 4-row table.]
- **Key data issues and treatments:** [e.g. structural missingness of fields introduced after 20XX; outlier caps; excluded post-origination fields.]

## 3. Methodology

- **Features:** [N final characteristics from M candidates; selection criteria (IV, correlation, sign, stability, business sense).]
- **Binning / transformation:** [method, monotonic constraints, min bin size, missing handling.]
- **Model:** [logistic regression on WoE; regularisation if any.]
- **Scaling:** [PDO = …, base score … at odds …:1.]
- **Calibration:** [method, fitted on which sample.]
- **Challenger:** [LightGBM configuration in one line; role = benchmark.]
- **Decision rule:** [cutoff = …; rationale: expected-profit maximisation subject to …]

## 4. Performance

| Model | Sample | AUC | Gini | KS | Brier | Mean PD / observed bad rate |
|---|---|---|---|---|---|---|
| Scorecard | dev_train | | | | | |
| Scorecard | OOT | | | | | |
| LightGBM | OOT | | | | | |
| `sub_grade` benchmark | OOT | | | — | — | — |

[Two sentences interpreting: stability dev → OOT, champion vs challenger gap with CI, calibration quality.]
**At chosen cutoff (OOT):** approval rate […]%, bad rate among approved […]%, expected profit […] per application.

## 5. Limitations

- [Accepted-only data → selection bias; no reject inference.]
- [Right-censoring / outcome maturity assumptions.]
- [Profit model simplifications (LGD, revenue, no time value of money).]
- [Scope: 36-month loans only; US marketplace lender; period-specific economy.]
- [Any feature with known instability or fairness concern.]

## 6. Key assumptions

- [Features are known and unchanged at `issue_d`.]
- [Future applicant population resembles 20XX–20XX unless monitoring shows otherwise.]
- [LGD = …, revenue = …, cost of funds = …]
- [Any other material assumption from the decision log.]

## 7. Monitoring plan

| Metric | Frequency | Green | Amber | Red | Action on Red |
|---|---|---|---|---|---|
| Score PSI vs dev | Monthly | < [0.10] | [0.10–0.25] | ≥ [0.25] | [investigate; recalibrate/redevelop per trigger rule] |
| Characteristic CSI | Monthly | | | | |
| Gini (mature vintages) | Quarterly | | | | |
| Calibration: mean PD vs bad rate | Quarterly | | | | |
| Approval rate / bad rate at cutoff | Monthly | | | | |

**Retrain trigger:** [exact rule, e.g. "score PSI ≥ 0.25 in 2 consecutive months OR ≥ 2 key characteristics CSI ≥ 0.25 OR relative Gini drop ≥ 10% vs development".]

## 8. Sign-off (simulated)

| Role | Name | Date | Decision |
|---|---|---|---|
| Developer | | | |
| Independent reviewer | [e.g. peer / mentor] | | |
