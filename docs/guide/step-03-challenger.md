# Step 3: Challenger Model (LightGBM)

**Time:** 3 h core + 2 h stretch · **Milestone:** M3 (with Step 4) · **Notebook:** `04_challenger_lgbm` · **Module:** `challenger.py` · **Requirements:** FR-3.1–3.5

---

## 1. Goal

Train a LightGBM challenger on **exactly the same rows and candidate features** as the scorecard, and answer one business question:

> *How much discriminatory power does the transparent scorecard give up, and is that gap worth the loss of transparency?*

## 2. Why it matters in bank credit risk

Banks run **champion/challenger** comparisons to justify model choices to validators and committees:
- If a gradient-boosting model is only marginally better, the scorecard wins on transparency, stability and ease of monitoring.
- If the gap is large, it signals missing features or interactions the scorecard should capture. Sometimes the challenger *becomes* the champion, with extra explainability controls.

Either way, you need a fair, honest comparison.

## 3. Key concepts

| Concept | In one breath |
|---|---|
| **Gradient-boosted trees** | Many shallow trees added one at a time, each fitting the errors of the previous ones. They capture non-linearities and interactions automatically |
| **Early stopping** | Stop adding trees when the validation metric stops improving. The validation set therefore *influences* the model, so it can't be your final test |
| **Key LightGBM knobs** | `num_leaves`, `min_child_samples` (complexity); `learning_rate` + number of trees; `feature_fraction` / `bagging_fraction` (randomness); `lambda_l2` (regularisation) |
| **Native categorical / missing handling** | LightGBM can split on categories and route missing values without WoE. That's convenient, but high-cardinality categoricals overfit easily |
| **Monotone constraints** | Force the prediction to only rise (or fall) with a feature. You trade a little accuracy for behaviour a validator can sign off on |
| **Fair comparison** | Same rows, same target, same metric code, same samples. Otherwise the gap is meaningless |
| **Uncertainty of a metric** | An AUC difference of 0.005 on 100k rows may or may not be real. Bootstrap the OOT sample to get a confidence interval |
| **Feature importance ≠ explanation** | Split counts and gain describe how the model was *built*, not why one applicant was declined. That's Step 4 |

**Resources**
- LightGBM "Parameters Tuning" guide: <https://lightgbm.readthedocs.io/en/stable/Parameters-Tuning.html>
- Ke, G. et al. (2017). "LightGBM: A Highly Efficient Gradient Boosting Decision Tree." *NeurIPS*.

## 4. Tasks (in order)

1. Prepare features from the **pre-binning** candidate list: raw numerics, low-cardinality categoricals as `category` dtype. Use exactly the same rows as the scorecard.
2. Train a **baseline** with sensible defaults, early stopping on `dev_valid`, `seed` from config, and deterministic settings. Record the best iteration.
3. **Light tuning:** a small random search (≤ 20 trials) over 4–5 knobs, scored on `dev_valid` AUC. Keep a results table.
4. **Evaluate** the final model on dev_train, dev_valid and OOT with the **same `metrics.py` functions** as Step 2.
5. Build the **champion vs challenger table** (AUC, Gini, KS per sample, plus the train → OOT drop for each).
6. Look at the **overfitting gap** (dev_train vs OOT) for both models. Which is more stable?
7. Write a 3–5 sentence **recommendation** in the notebook: keep the scorecard, switch, or improve the scorecard using what the challenger found (e.g. an interaction).
8. Save the model + metrics (`models/lgbm.joblib`, `reports/metrics/lgbm.json`), then update PROGRESS.md.
9. *Stretch:* bootstrap CI (e.g. 500 resamples of OOT) for Gini(challenger) − Gini(scorecard).
10. *Stretch:* add monotone constraints for 3–5 key features and report the Gini cost.

## 5. Acceptance criteria

- [ ] Scorecard and challenger evaluated on identical rows (assert equal indexes / ids)
- [ ] Early stopping and tuning used `dev_valid` only; OOT was used once, at the end
- [ ] Rerunning training gives the same OOT AUC to 4 decimals
- [ ] Comparison table recorded in PROGRESS.md and saved in `reports/metrics/`
- [ ] Written recommendation that references the size of the gap and stability, not just "LightGBM is better"
- [ ] Notebook passes Restart & Run All

## 6. Hints

### Task 2: reproducible LightGBM

<details><summary>Hint 1 (nudge)</summary>

LightGBM uses multithreading and random sampling. Which parameters control the randomness, and is a seed alone enough for identical results?
</details>

<details><summary>Hint 2 (approach)</summary>

Read the docs for the `seed`, `deterministic` and `force_row_wise` / `force_col_wise` parameters. Pass the validation set and an early-stopping callback. Fit on dev_train, validate on dev_valid.
</details>

<details><summary>Hint 3 (near-solution)</summary>

```
params = {objective: binary, metric: auc, seed: SEED, deterministic: True,
          force_row_wise: True, learning_rate: ~0.05, num_leaves: ~31,
          min_child_samples: large-ish (credit data is noisy), feature_fraction: ~0.8}
model = train(params, dev_train_set, valid_sets=[dev_valid_set],
              callbacks=[early_stopping(patience), log_evaluation(period)])
best_iteration = model.best_iteration
```
</details>

### Task 9: bootstrap CI for the Gini difference

<details><summary>Hint 1 (nudge)</summary>

To compare two models fairly, each bootstrap resample must score the *same* resampled rows with *both* models.
</details>

<details><summary>Hint 2 (near-solution)</summary>

```
rng = Generator(SEED)
diffs = []
repeat B times:
    idx = rng.integers(0, n_oot, n_oot)          # resample rows with replacement
    diffs.append(gini(y[idx], p_lgbm[idx]) - gini(y[idx], p_sc[idx]))
CI = percentiles 2.5 and 97.5 of diffs           # does it include 0?
```
</details>

## 7. Common pitfalls

- **Early stopping on OOT** (or tuning on it). OOT is then no longer out-of-time.
- **`is_unbalance` / `scale_pos_weight`** improve nothing for ranking and **distort probabilities**, which breaks Step 4 and Step 5. Leave them off unless you have a reason and recalibrate afterwards.
- **High-cardinality categoricals** (`emp_title`, `zip_code`) memorise noise. They should already be excluded from Step 1.
- **Comparing on different rows**, e.g. LightGBM keeps rows the scorecard pipeline dropped.
- **"Better AUC on dev_train"** says nothing. Compare on OOT, and compare the *drop*.
- **Endless tuning.** Gains beyond the first few trials are usually smaller than the noise. Time-box it.

## 8. Self-check questions

1. LightGBM beats the scorecard by 2 Gini points on OOT. Should the bank switch? List what else you need to know.
2. Why must early stopping not use the OOT set, even though it "only picks the number of trees"?
3. What's the difference between split and gain importance, and why is neither a decline reason?
4. How would you show that an AUC difference between two models is not just noise?
5. What do monotone constraints buy you in a regulated setting, and what do they cost?
