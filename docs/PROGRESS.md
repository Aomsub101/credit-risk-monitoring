# Progress Tracker

Update this file at the end of every work session (2 minutes) and at the end of every step (10 minutes).
It is your lab notebook, your interview prep, and the raw material for the model documentation.

---

## 1. Status board

Status: ⬜ not started · 🟨 in progress · ✅ done · ⏸ blocked

| Step | Status | Started | Finished | Hours (plan / actual) | Headline result |
|---|---|---|---|---|---|
| 0 Setup | ⬜ | | | 1.5 / | |
| 1 EDA & data quality | ⬜ | | | 6 / | |
| 2 Classic scorecard | ⬜ | | | 7 / | |
| 3 Challenger | ⬜ | | | 3 / | |
| 4 Calibration & explainability | ⬜ | | | 4 / | |
| 5 Business layer | ⬜ | | | 3 / | |
| 6 Monitoring | ⬜ | | | 4 / | |
| 7 Ship it | ⬜ | | | 5 / | |

## 2. Data provenance

| Item | Value |
|---|---|
| Source | Kaggle `wordsforthewise/lending-club` |
| File | `accepted_2007_to_2018Q4.csv.gz` |
| Downloaded on | |
| SHA-256 | |
| Raw rows × columns | |
| Data dictionary | `LCDataDictionary.xlsx` (source: ) |

## 3. Population & sample sizes

### Filter waterfall (Step 1)

| # | Filter | Rows remaining | Bad rate |
|---|---|---|---|
| 0 | Raw file | | |
| 1 | | | |
| 2 | | | |
| 3 | | | |

### Final samples

| Sample | Issue period | Rows | Bads | Bad rate |
|---|---|---|---|---|
| dev_train | | | | |
| dev_valid | | | | |
| oot | | | | |
| monitor | | | n/a (immature) | |

## 4. Metrics board

Fill these from `reports/metrics/*.json`. Never type a number by hand that the code didn't produce.

| Model | Sample | AUC | Gini | KS | Brier | Mean PD vs bad rate |
|---|---|---|---|---|---|---|
| `sub_grade` benchmark | OOT | | | | — | — |
| Scorecard | dev_train | | | | | |
| Scorecard | dev_valid | | | | | |
| Scorecard | OOT | | | | | |
| LightGBM | dev_train | | | | | |
| LightGBM | dev_valid | | | | | |
| LightGBM | OOT | | | | | |
| Scorecard (calibrated) | OOT | | | | | |
| LightGBM (calibrated) | OOT | | | | | |

| Business / monitoring | Value |
|---|---|
| Chosen cutoff (score / PD) | |
| Approval rate at cutoff (OOT) | |
| Bad rate among approved (OOT) | |
| Expected profit per application | |
| Max monthly score PSI 2016–2018 | |
| First month trigger fired (real data) | |
| Synthetic shock detected? | |

## 5. Decision log

Every non-obvious choice goes here. A model validator will ask "why?" about each one.

| Date | Step | Decision | Alternatives considered | Reason | Req. ID |
|---|---|---|---|---|---|
| 2026-10-29 | 0 | Use Lending Club accepted loans (Kaggle `wordsforthewise/lending-club`) | Home Credit, Give Me Some Credit | Only candidate with a real origination date (`issue_d`) → honest OOT split and real drift | DR-1 |
| 2026-10-29 | 0 | uv for environment, Streamlit for demo | Poetry/pip; FastAPI | Lockfile reproducibility; Streamlit fastest for a visual demo | TR-1 |
| | 1 | Population: | | | DR-6 |
| | 1 | Target definition: | | | DR-6 |
| | 1 | Time windows: | | | DR-7 |
| | 1 | `grade`/`sub_grade`/`int_rate`/`installment`: | | | |
| | 2 | Scaling (PDO / base score / base odds): | | | FR-2.5 |
| | 4 | Calibration method: | | | FR-4.2 |
| | 5 | Cutoff: | | | FR-5.3 |
| | 6 | PSI/CSI thresholds and trigger rule: | | | FR-6.3 |

## 6. Self-check answers

Write short answers to each guide's self-check questions here. These are your interview answers.

### Step 1
1.
2.
3.
4.
5.

### Step 2
1.
2.
3.
4.
5.

### Step 3

### Step 4

### Step 5

### Step 6

### Step 7

## 7. Session log

Newest first. Keep each entry to 3–5 lines.

```
### YYYY-MM-DD (Xh) Step N
Did:
Learned:
Blocked by / questions:
Next:
```

## 8. Open questions for my mentor

-
