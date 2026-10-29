# Step 7: Ship It (Pipeline, Tests, Streamlit, README, Model Documentation)

**Time:** 5 h core + 3 h stretch · **Milestone:** M5 → `v1.0.0` · **Files:** `scripts/run_pipeline.py`, `tests/`, `app/streamlit_app.py`, `README.md`, `docs/MODEL_DOCUMENTATION.md` · **Requirements:** FR-7.1–7.6, full quality checklist

---

## 1. Goal

Make the project **reproducible by a stranger** and **understandable in 2 minutes**:
- one command rebuilds everything,
- tests prove the maths,
- a demo shows a decision with reasons,
- a README and a 1-page model document tell the story with real numbers.

## 2. Why it matters in bank credit risk

A model that can't be reproduced, tested and explained doesn't pass validation, however good its Gini. For your career, this step turns "I did a Kaggle notebook" into "I built and documented a model the way a bank does". Hiring managers and validators read the README and the model document first, and often only those.

## 3. Key concepts

| Concept | In one breath |
|---|---|
| **Idempotent pipeline** | Running it twice gives identical outputs. No hidden state, no manual notebook steps |
| **Artifacts vs code** | Code is committed. Artifacts (models, processed data) are regenerated from code + raw data. Metadata (window, features, versions, commit hash) travels with each artifact |
| **Tests for analytics** | Test the *maths* on tiny hand-made inputs (KS, WoE, PSI, scaling) and test the *guards* (no leakage columns in feature lists), not "the model's AUC is 0.70" |
| **Streamlit caching** | `st.cache_resource` for models (load once), `st.cache_data` for data frames. The app should never train |
| **Model documentation** | A short, factual record: purpose, data, method, performance, limitations, monitoring. It's the bank version of a model card |

**Resources**
- Streamlit docs, especially "Caching overview" and "Deploy": <https://docs.streamlit.io/>
- Mitchell, M. et al. (2019). "Model Cards for Model Reporting." *FAT\* '19*. Read it alongside your [MODEL_DOCUMENTATION_TEMPLATE.md](../MODEL_DOCUMENTATION_TEMPLATE.md).

## 4. Tasks (in order)

1. **Pipeline:** `scripts/run_pipeline.py` runs, in order: verify raw SHA-256 → make dataset → fit binning + scorecard → fit LightGBM → fit calibrators → choose cutoff → write metrics JSON → run monitoring for all months. Each stage calls `src/` functions; the notebooks are *not* part of the pipeline.
2. **Reproducibility check:** run the pipeline twice and diff `reports/metrics/*.json`. They must be identical and must match PROGRESS.md.
3. **Tests:** fill `tests/` (`test_metrics`, `test_binning`, `test_scorecard_scaling`, `test_monitoring`, `test_no_leakage`). `uv run pytest` must be green. Run `uv run ruff check .` and fix what it finds.
4. **App sample:** write ≤ 500 OOT rows (only the columns the app needs) to `app/sample_applicants.parquet`. It's public data and small, so commit it.
5. **Streamlit app** (`app/streamlit_app.py`), three tabs:
   - *Score an applicant:* pick a sample applicant (and optionally edit 2–3 fields) → score, calibrated PD, approve/decline vs cutoff, top reasons.
   - *Model performance:* metrics table + ROC/KS or gains chart from `reports/`.
   - *Monitoring:* PSI over time, CSI heatmap, latest RAG status.
6. **Model documentation:** copy the template to `docs/MODEL_DOCUMENTATION.md` and fill it in. **One page.** Numbers from `reports/metrics/` only.
7. **README:** replace every `TBD` marker. Add the results table, 2–3 figures (strategy curve, PSI chart, app screenshot), how to run, limitations.
8. **Final review:** walk through the quality checklist in [REQUIREMENTS.md](../REQUIREMENTS.md#5-quality-checklist), and ask your mentor for a full review.
9. **Release:** PR `dev` → `main` with a conventional title, merge, tag `v1.0.0`, push the tag.
10. *Stretch:* deploy on Streamlit Community Cloud (check its docs for supported dependency files; `uv export --format requirements-txt` can produce one if needed).
11. *Stretch:* GitHub Actions workflow running `ruff` + `pytest` on push (look at the `astral-sh/setup-uv` action).

## 5. Acceptance criteria

- [ ] Fresh clone + raw file → `uv sync --locked` → `uv run python scripts/run_pipeline.py` reproduces PROGRESS.md metrics to 3 decimals
- [ ] `uv run pytest` green; `uv run ruff check .` clean
- [ ] `uv run streamlit run app/streamlit_app.py` works **without** the raw data file present
- [ ] App shows score, PD, decision and ≤ 4 readable reasons for any sample applicant
- [ ] `docs/MODEL_DOCUMENTATION.md` fits on one page; every number traceable to `reports/metrics/`
- [ ] README: no `TBD` left; "How to run" ≤ 5 commands; limitations section present
- [ ] Every quality checklist box ticked
- [ ] `v1.0.0` tag on `main`

## 6. Hints

### Task 1: pipeline structure

<details><summary>Hint 1 (nudge)</summary>

If your notebooks already call `src/` functions, the pipeline is mostly a list of those calls in order. Which steps still live *only* in a notebook?
</details>

<details><summary>Hint 2 (approach)</summary>

One function per stage, each taking paths/config and writing its outputs, plus a `main()` that calls them in order and logs timing. Fail fast if the raw checksum doesn't match. Save metadata next to every model.
</details>

<details><summary>Hint 3 (near-solution)</summary>

```
def main():
    check_sha256(RAW_FILE, EXPECTED_SHA256)
    splits     = make_dataset()                  # -> data/processed/*
    sc         = train_scorecard(splits)         # -> models/, reports/scorecard_table.csv
    lgbm       = train_challenger(splits)
    calibrators = fit_calibrators(splits, sc, lgbm)
    cutoff     = choose_cutoff(splits, calibrators)
    write_metrics(...)                           # -> reports/metrics/*.json
    for month in monitoring months: run_monitoring(month)
if __name__ == "__main__": main()
```
</details>

### Task 3: a leakage guard test

<details><summary>Hint 1 (nudge)</summary>

You already have `LEAKAGE_COLUMNS` and the final feature lists in `src/`. What single assertion would have caught a leakage bug in Step 1?
</details>

### Task 5: app structure

<details><summary>Hint 1 (nudge)</summary>

The app needs to load the binning, scorecard, calibrator, cutoff and reason-code table. Everything else is display. Where should loading happen so it runs only once?
</details>

<details><summary>Hint 2 (approach)</summary>

One cached loader function returns all artifacts. One `score_applicant(row)` function, ideally in `src/` so it's tested, returns score, PD, decision and reasons. Tabs (`st.tabs`) keep it to a single file. Read figures and metrics from `reports/` rather than recomputing them.
</details>

## 7. Common pitfalls

- **The app retrains at startup** or needs the 392 MB raw file.
- **Absolute or backslash paths** that work only on your laptop.
- **Pickled models loaded with a different library version.** Record versions in metadata; the lockfile protects you.
- **Committing `models/` or `data/`.** Check `git status` before every commit.
- **README claims without numbers**, or numbers that don't match the reports.
- **A three-page "one-page" model document.** Cutting is part of the skill.
- **Unhandled inputs in the app** (missing value, unseen category). Your binning's missing/other bins should handle them, so test that.

## 8. Self-check questions

1. A teammate clones your repo. List the exact commands to reproduce your OOT Gini.
2. What would a model validator look at first in your documentation, and what's your weakest section?
3. What would have to change to deploy this inside a real bank (data, governance, infrastructure, monitoring ownership)?
4. Which single assumption in your project are you least confident about, and how would you test it with more time?
5. How does your app handle an input value it never saw in training?
