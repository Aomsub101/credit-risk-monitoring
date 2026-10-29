# Step 1: EDA, Data Quality, Leakage & Time-Based Split

**Time:** 6 h core + 2 h stretch · **Milestone:** M1 · **Notebooks:** `01_extract_data`, `02_eda_data_quality` · **Requirements:** FR-1.1–1.8, DR-5–DR-9

---

## 1. Goal

Turn the raw Lending Club file into a **clean, documented modelling dataset**:
- a defensible target and population,
- a feature list where every column is provably known at the application date,
- and four time-ordered samples (`dev_train`, `dev_valid`, `oot`, `monitor`) saved as parquet.

## 2. Why it matters in bank credit risk

Most model validation findings are about **data and definitions**, not algorithms. A wrong "bad" definition, a leaked variable, or a random split can make a weak model look excellent. Validators will reject it, and in production it will lose money. Regulators expect an out-of-time test because a credit model is always used on *future* applicants.

Everything in Steps 2–7 inherits the decisions you make here. Spend the time.

## 3. Key concepts

| Concept | In one breath |
|---|---|
| **Observation point** | The moment the decision is made: here `issue_d`. A feature is legal only if its value was known at that moment |
| **Performance window & target** | The period after the observation point in which we watch for "bad". Here: the life of a 36-month loan, ending in a final `loan_status` |
| **Indeterminates** | Loans whose outcome isn't settled yet (Current, Late, In Grace). They are neither good nor bad, so you exclude them from development and say so |
| **Right-censoring / maturity** | Recent vintages haven't had time to default, so their bad rate *looks* lower. It's a reporting artefact, not better borrowers |
| **Vintage analysis** | Bad rate grouped by origination month. It's the credit analyst's first chart |
| **Target leakage** | A feature that contains information from after the observation point (e.g. total payments received). It inflates every metric |
| **Model-on-model features** | `grade`, `sub_grade`, `int_rate`: Lending Club's *own* risk assessment. Known at issue date, but using them means your model leans on someone else's model |
| **Out-of-time (OOT) vs out-of-sample** | OOT = a later time period never used in development. A random hold-out from the same period is far less convincing |
| **Informative missingness** | In credit data, "missing" often *means* something. For example, `mths_since_last_delinq` is missing when there has *never* been a delinquency |
| **Structural missingness** | A column that didn't exist before some date is 100% missing for early vintages. That's a pipeline fact, not borrower behaviour |

**Resources**
- Siddiqi, N. (2017). *Intelligent Credit Scoring* (2nd ed.), Wiley. Read the chapters on development database creation and defining "bad".
- Kaufman, S., Rosset, S., Perlich, C. & Stitelman, O. (2012). "Leakage in Data Mining: Formulation, Detection, and Avoidance." *ACM TKDD* 6(4).

## 4. Tasks (in order)

**A. Extract (`01_extract_data.ipynb`)**

1. Read **only the header** of the raw file and the data dictionary. How many columns are there? Which ones have no description?
2. Load the file with an explicit column list and dtypes. Read `id` as a string at first (why? see pitfalls). Parse `issue_d` into a real date.
3. Find and drop non-loan rows (e.g. rows with no `issue_d` or `loan_amnt`). Record how many.
4. Save `data/interim/loans.parquet`. From now on, never read the CSV again.

**B. Audit & define (`02_eda_data_quality.ipynb`)**

5. **Column audit:** classify every one of the 151 columns: `application_time`, `post_origination`, `lc_risk_output`, `id_admin`, `text_high_cardinality`, `target_related`. Add keep/drop and a one-line reason. Save `reports/column_audit.csv`.
6. **Target & population:** list `loan_status` values with counts. Map them to bad / good / indeterminate. Choose the population (term, policy flag, joint apps). Build the **filter waterfall** (rows and bad rate after each filter).
7. **Vintage analysis:** loan volume and bad rate by issue month, split by term. Add the **share of indeterminates by issue year**. That chart tells you where outcomes are mature.
8. **Missingness:** % missing per kept column, then a **column × issue-year** heatmap. Mark each missing pattern as informative, structural, or random.
9. **Univariate checks:** distributions, impossible values and outliers for numeric features; levels and rare categories for categoricals. Write a treatment plan (cap? bin? flag?), but don't fit anything on the full data yet.
10. **Leakage screen:** single-feature AUC of each candidate on dev data. Investigate anything above ~0.80.
11. **Time-based split:** decide the windows from your vintage and maturity charts (draft: dev_train ≤ 2014-06, dev_valid 2014-07–12, OOT 2015, monitor 2016-01–2018-12). Save four parquet files, assert there's no `id` overlap, and print rows and bad rate per split.

**C. Consolidate**

12. Promote the load / target / population / split logic to `src/credit_risk/data.py`, and the drop lists to `src/credit_risk/features.py` (`LEAKAGE_COLUMNS`, `CANDIDATE_FEATURES`). Re-import them in the notebook.
13. Write the **data quality summary** (last notebook cell): top findings and treatments. Fill in the waterfall, samples and decision log in `PROGRESS.md`.
14. *Stretch:* add `credit_history_months` from `earliest_cr_line` and `issue_d`; vintage bad rate by `grade`; a README-ready chart of volume vs bad rate over time.

## 5. Acceptance criteria

- [ ] `reports/column_audit.csv` has exactly one row per raw column, and none are left unclassified
- [ ] `LEAKAGE_COLUMNS` ∩ `CANDIDATE_FEATURES` = ∅ (assert in the notebook; it becomes a test in Step 7)
- [ ] Target mapping and population filters live in `src/credit_risk/data.py`; the waterfall is in PROGRESS.md
- [ ] Vintage chart + indeterminate-share chart exist, with a written conclusion about where outcomes are mature
- [ ] Missingness-by-year heatmap exists; every column with > 20% missing has a stated reason and treatment
- [ ] Four parquet files exist; no `id` overlap; date ranges and bad rates recorded
- [ ] OOT has no indeterminates in the modelling target (or their exclusion is documented)
- [ ] Both notebooks pass Restart & Run All
- [ ] Decisions on population, target, windows and `grade`/`sub_grade`/`int_rate` logged

## 6. Hints

### Task 2: loading a large file

<details><summary>Hint 1 (nudge)</summary>

Loading all 151 columns of 2.26M rows with default dtypes can take several GB of RAM. How many columns do you actually need, and what's the smallest dtype for each?
</details>

<details><summary>Hint 2 (approach)</summary>

First read with `nrows=0` to get the header. Decide your `usecols` after a first pass of the audit, but keep everything you need for the audit and for Step 5's profit back-test (repayments are fine as *outcomes*). Use `category` for low-cardinality strings, `float32` where precision doesn't matter, and `format=` when parsing `issue_d` (look at a few raw values to find the pattern).
</details>

<details><summary>Hint 3 (near-solution)</summary>

```
header   = read_csv(RAW_FILE, nrows=0).columns
usecols  = [c for c in header if c in KEEP_FOR_NOW]
dtypes   = {"id": "string", <low-card cols>: "category", <numerics>: "float32"}
df       = read_csv(RAW_FILE, usecols=usecols, dtype=dtypes, low_memory=False)
df       = drop rows where issue_d or loan_amnt is missing      # log the count
df["issue_d"] = to_datetime(df["issue_d"], format=<pattern you found>)
df.to_parquet(INTERIM_DIR / "loans.parquet")
```
</details>

### Task 5: column audit

<details><summary>Hint 1 (nudge)</summary>

For each column, ask: *"On the day the loan was issued, would Lending Club have known this value?"*
</details>

<details><summary>Hint 2 (approach)</summary>

Read the dictionary description, don't guess from the name. Words that usually signal **post-origination**: *last…, total received, recoveries, outstanding principal, next payment, settlement, hardship*. Lending Club's own pricing outputs (`grade`, `sub_grade`, `int_rate`, `installment`) get their own category, so the decision about them is explicit. Free text and very high-cardinality fields (`emp_title`, `desc`, `title`, `url`) are a separate category too.
</details>

<details><summary>Hint 3 (near-solution)</summary>

```
audit = table(column, description from dictionary)
audit["category"] = your rule-based first guess (keyword patterns)
# then MANUALLY review every row; rules are only a starting point
audit["keep"]   = category == "application_time" and not (geo or free text)
audit["reason"] = one line per column
save audit; build LEAKAGE_COLUMNS from category in {post_origination, target_related}
```
</details>

### Task 6: target and population

<details><summary>Hint 1 (nudge)</summary>

`value_counts()` on `loan_status`. Which values are *final* outcomes, and which describe a loan that's still running?
</details>

<details><summary>Hint 2 (approach)</summary>

Map final losses → 1, full repayment → 0, still-running → missing (indeterminate). Notice the two `"Does not meet the credit policy. Status:…"` values: they come from an older policy period. Decide whether they belong in your population. Do the same for 60-month loans (can their outcomes mature within this data?) and for joint applications.
</details>

<details><summary>Hint 3 (near-solution)</summary>

```
BAD_STATUSES  = {statuses meaning the loan was lost}
GOOD_STATUSES = {statuses meaning fully repaid}
target = 1 if status in BAD_STATUSES else 0 if status in GOOD_STATUSES else NA
waterfall = []
for name, mask in ordered population filters:
    df = df[mask]; waterfall.append((name, len(df), df.target.mean()))
```
</details>

### Task 11: time-based split

<details><summary>Hint 1 (nudge)</summary>

Look at your indeterminate-share-by-year chart. In which year do unresolved loans start to be a big share? That's roughly where your labelled data must end.
</details>

<details><summary>Hint 2 (approach)</summary>

Split purely on `issue_d` boundaries kept in `config.py`. Development must come before OOT, which must come before monitor. Hold out the last few development months as `dev_valid`, so that tuning, early stopping and calibration never touch OOT. Check every split has enough bads (hundreds at least, preferably thousands).
</details>

<details><summary>Hint 3 (near-solution)</summary>

```
dev_train = df[issue_d <= DEV_TRAIN_END]
dev_valid = df[DEV_TRAIN_END < issue_d <= DEV_END]
oot       = df[OOT_START <= issue_d <= OOT_END]
monitor   = population (before target filtering) with MONITOR_START <= issue_d <= MONITOR_END
assert no shared ids between any two; print n, bads, bad rate per split; save each to parquet
```
</details>

## 7. Common pitfalls

- **Random `train_test_split`.** It hides time drift and overstates performance.
- **Keeping `Current` loans as "good".** A loan that hasn't defaulted *yet* isn't good.
- **Imputing `mths_since_last_delinq` NaN with 0.** 0 means "delinquent last month", the opposite of what NaN means here.
- **Fitting caps, imputers or encoders on all data.** Plan treatments now, but fit them on `dev_train` only (Step 2).
- **Treating structural missingness as signal.** A field that only exists from a certain year will look "predictive of time", not of risk.
- **Using `zip_code` / `addr_state` without thinking.** Geography can proxy for protected characteristics (fair-lending risk).
- **pandas 3 Copy-on-Write.** `df[col][mask] = value` silently does nothing. Use `df.loc[mask, col] = value`.
- **Whitespace and formats.** `term` may have leading spaces, `emp_length` is text (`"10+ years"`), and some versions store rates as strings with `%`. Check before you convert.
- **`id` as integer.** Footer or summary rows with text in `id` will crash or corrupt an int parse.

## 8. Self-check questions

1. Why is an out-of-time test more convincing than a random 80/20 split for a credit model? What specific risk does it catch?
2. Bad rate is about 18% for 2014 vintages and about 9% for 2017 vintages. Did credit quality improve? Give two other explanations.
3. Name three columns you excluded as leakage and explain how each would inflate performance.
4. Should `sub_grade` be an input to your scorecard? Argue both sides, then state your decision.
5. What does a missing `mths_since_last_delinq` mean, and how will your binning treat it?
