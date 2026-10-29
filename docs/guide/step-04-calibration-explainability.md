# Step 4: Calibration & Explainability

**Time:** 4 h core + 2 h stretch · **Milestone:** M3 · **Notebook:** `05_calibration_explain` · **Modules:** `calibration.py`, `explain.py` · **Requirements:** FR-4.1–4.6

---

## 1. Goal

1. Turn scores into **trustworthy probabilities of default** and prove it with reliability diagrams and Brier scores on OOT.
2. Explain the models **globally** (what drives risk overall) and **locally** (why *this* applicant).
3. Produce **plain-language decline reasons** a customer could read.

## 2. Why it matters in bank credit risk

- **Ranking is not enough.** Pricing, expected loss, provisioning and the profit cutoff in Step 5 all multiply by PD. A model that ranks well but says 6% when reality is 12% will mis-set the cutoff and under-provision.
- **Explanations are a legal requirement in many markets.** For example, US ECOA / Regulation B requires creditors to give declined applicants the *specific principal reasons*. The CFPB has stated (Circular 2022-03) that using a complex algorithm is no excuse for vague reasons.
- **Validators use global explanations** to check that the model's logic matches credit sense.

## 3. Key concepts

| Concept | In one breath |
|---|---|
| **Discrimination vs calibration** | Discrimination = ranking (AUC). Calibration = predicted PD ≈ observed bad rate. These are independent: a model can have one without the other |
| **Reliability diagram** | Bin predictions (e.g. 10 bins), then plot mean predicted PD vs observed bad rate per bin. Perfect calibration falls on the diagonal |
| **Brier score** | Mean of (PD − outcome)². Lower is better. It mixes calibration and discrimination |
| **Calibration-in-the-large** | Mean predicted PD vs overall observed bad rate. It's the first thing to check on OOT, because the portfolio bad rate drifts over time |
| **Platt scaling vs isotonic** | Platt fits a logistic curve on the score (2 parameters: smooth, robust). Isotonic fits a free monotone step function (flexible, needs more data, can overfit) |
| **Where to fit a calibrator** | On data the model was *not* trained on (`dev_valid`), then judge it on OOT. Monotone calibration doesn't change AUC |
| **SHAP values** | Additive per-feature contributions that sum (with a base value) to the model's output for one applicant. For LightGBM binary models they're in **log-odds** units |
| **Points-below-max reasons** | For a scorecard: for each characteristic, the gap between the maximum possible points and the applicant's points. The biggest gaps are the principal reasons |
| **Reason code mapping** | A fixed table: feature (+ direction) → customer-readable sentence. Common practice is up to 4 reasons (Regulation B commentary notes that more than four is unlikely to help) |

**Resources**
- scikit-learn User Guide, "Probability calibration": <https://scikit-learn.org/stable/modules/calibration.html>
- Molnar, C. *Interpretable Machine Learning*, chapter on SHAP: <https://christophm.github.io/interpretable-ml-book/>

## 4. Tasks (in order)

1. **Diagnose:** for scorecard and LightGBM, on dev_valid and OOT, produce reliability diagrams, Brier score and mean PD vs observed bad rate. Write down what you see *before* fixing anything.
2. **Calibrate:** fit Platt and isotonic calibrators on `dev_valid` model outputs, and evaluate both on OOT. Pick one per model and log why.
3. **Sanity check:** assert that OOT AUC is unchanged by calibration (to ~4 decimals).
4. **Global SHAP** for LightGBM on a sample (e.g. 5–10k OOT rows): mean |SHAP| ranking and a beeswarm plot. Compare the top 10 with the scorecard's top characteristics by IV or points range. Where do they disagree?
5. **Dependence plots** for 2 features. Does the shape match your scorecard bins?
6. **Reason-code table** (`reports/reason_codes.csv`): for each final characteristic, the plain-language sentence used when it hurts the applicant (e.g. "Revolving credit utilisation is high").
7. **Scorecard reasons:** implement points-below-max in `src/credit_risk/explain.py` → top 4 reasons per applicant. Unit-test it on a toy 2-characteristic scorecard.
8. **Worked examples:** pick 3 OOT applicants (clear approve, clear decline, borderline; use a provisional cutoff) and write a short "decision letter" for each: score, PD, decision, reasons. Save to `reports/decision_examples.md`.
9. Save calibrator(s) to `models/`, then update PROGRESS.md (Brier, calibration-in-the-large, chosen method).
10. *Stretch:* SHAP-based reasons for LightGBM (top features pushing towards bad), and the % agreement of top-2 reasons with the scorecard on 1,000 applicants.

## 5. Acceptance criteria

- [ ] Reliability diagrams (before/after) for both models on OOT saved to `reports/figures/`
- [ ] Calibrators fit on `dev_valid` only; OOT Brier and mean-PD gap reported before and after
- [ ] AUC unchanged by calibration (asserted)
- [ ] SHAP global plot + comparison with the scorecard written up in 3–5 sentences
- [ ] `explain.py` returns ≤ 4 reasons per applicant using customer-readable text; test passes
- [ ] 3 worked examples in `reports/decision_examples.md`
- [ ] Notebook passes Restart & Run All

## 6. Hints

### Task 2: calibrating

<details><summary>Hint 1 (nudge)</summary>

A calibrator is just a small model that maps "raw model output" → "probability", trained on (output, actual outcome) pairs. Which sample should those pairs come from?
</details>

<details><summary>Hint 2 (approach)</summary>

The most transparent route, and one that works identically for the scorecard and LightGBM: take the model's output on `dev_valid`. Fit a 1-feature logistic regression on it (Platt) and an `IsotonicRegression` (isotonic). Apply both to OOT outputs. If you prefer `CalibratedClassifierCV` for an already-fitted model, check the docs for your installed scikit-learn version, because the "prefit" API changed recently (`FrozenEstimator`).
</details>

<details><summary>Hint 3 (near-solution)</summary>

```
s_valid = model output on dev_valid (log-odds or score);  y_valid
platt    = fit 1-feature logistic regression(s_valid -> y_valid)
iso      = fit IsotonicRegression(out_of_bounds="clip")(p_valid -> y_valid)
for each calibrator: p_oot = calibrator(model output on OOT)
    report brier(y_oot, p_oot), mean(p_oot) vs mean(y_oot), reliability bins
assert auc(y_oot, p_oot_calibrated) ≈ auc(y_oot, p_oot_raw)
```
</details>

### Task 7: points-below-max reasons

<details><summary>Hint 1 (nudge)</summary>

For each characteristic, what's the best score an applicant *could* have earned? How far below that is this applicant?
</details>

<details><summary>Hint 2 (approach)</summary>

From the scorecard table, compute max points per characteristic once. For an applicant, the shortfall per characteristic = max − earned. Sort descending, take the top 4 with shortfall > 0, and map to text via `reason_codes.csv`.
</details>

<details><summary>Hint 3 (near-solution)</summary>

```
max_pts = scorecard_table.groupby(characteristic).points.max()
def reasons(applicant_points: dict[characteristic -> points], k=4):
    shortfall = {c: max_pts[c] - applicant_points[c] for c in applicant_points}
    top = characteristics sorted by shortfall desc, keep shortfall > 0, first k
    return [REASON_TEXT[c] for c in top]
```
</details>

## 7. Common pitfalls

- **Calibrating on training data.** It looks perfect and is meaningless.
- **Calibrating on OOT and then reporting OOT calibration.** That's circular.
- **Isotonic on small samples** gives jagged, overfit steps. Compare it with Platt on OOT before choosing.
- **Reading SHAP for LightGBM as probabilities.** By default they're log-odds contributions.
- **Explaining the whole of OOT with SHAP.** It's slow and adds nothing; a sample is enough for global plots.
- **Reasons nobody can understand** ("WoE_revol_util low"), reasons the applicant can't act on, or reasons based on excluded or protected attributes.
- **Treating SHAP as causal.** It explains the model, not the world. Correlated features split credit unpredictably.

## 8. Self-check questions

1. Your model's AUC is 0.70 but it predicts an average PD of 8% while the observed OOT bad rate is 14%. Is it a good model? What do you do?
2. Why does a monotone calibration leave AUC unchanged?
3. Walk through how a points-based scorecard produces decline reasons for one applicant.
4. SHAP says "high annual income increased risk" for one applicant. How can that happen, and what would you tell a validator?
5. Why might a regulator be uneasy about SHAP-based reasons from a black-box model, even if they're "accurate"?
