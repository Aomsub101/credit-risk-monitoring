# Step 6: Monitoring (PSI/CSI, Drift Reports, Retrain Trigger)

**Time:** 4 h core + 2 h stretch · **Milestone:** M4 · **Notebook:** `07_monitoring` · **Module:** `monitoring.py` · **Script:** `scripts/monitor.py` · **Requirements:** FR-6.1–6.6, DR-9

---

## 1. Goal

Build the monitoring a bank would run every month after go-live:
- **score stability (PSI)** and **characteristic stability (CSI)** vs development, on real 2016–2018 applications,
- a **synthetic shock** to prove the alarms work,
- an explicit **trigger rule**,
- an auto-generated **monthly report**.

## 2. Why it matters in bank credit risk

Models decay. Marketing campaigns, new channels, policy changes and the economy all shift who applies. Supervisory guidance (e.g. SR 11-7) expects **ongoing monitoring with predefined thresholds and actions**.

In practice, defaults take months or years to appear, so **input drift is your early warning system**. You see the population change long before you can measure a drop in Gini. The 2016–2018 data mimics exactly that: you can see the applicants, but most outcomes aren't mature yet.

## 3. Key concepts

| Concept | In one breath |
|---|---|
| **PSI** | Σ over bins of (actual% − expected%) × ln(actual% / expected%), with bins fixed from **development**. It's symmetric, and 0 means identical distributions |
| **Rule-of-thumb thresholds** | < 0.10 stable · 0.10–0.25 moderate shift, investigate · ≥ 0.25 major shift, act. These are conventions, not statistical tests |
| **CSI** | The same formula applied to each characteristic's scorecard bins. It tells you *which* input moved |
| **Characteristic analysis (points shift)** | Σ (actual% − expected%) × points per bin. It tells you how many score points each characteristic's shift is worth |
| **Data drift vs concept drift** | Data drift = inputs change (PSI sees it). Concept drift = the relationship between inputs and default changes (only outcome monitoring sees it, and only with a lag) |
| **Outcome maturity lag** | To measure Gini on a vintage you need its outcomes. For 36-month loans, that takes years. Early-read metrics are biased by censoring |
| **Recalibrate vs retrain vs redevelop** | Adjust the PD mapping only · refit the same design on new data · rebuild variables and binning from scratch. The scale of the action should match the evidence |
| **RAG status** | Red/Amber/Green per metric per period, with persistence rules (e.g. Red only if breached two months running) to avoid alert fatigue |

**Resources**
- Yurdakul, B. (2018). *Statistical Properties of Population Stability Index*. Doctoral dissertation, Western Michigan University.
- Siddiqi, N. (2017). *Intelligent Credit Scoring* (2nd ed.). Read the chapter on scorecard monitoring (stability and characteristic analysis).

## 4. Tasks (in order)

1. **Implement `psi(expected, actual, edges)`** and **`csi(...)`** in `src/credit_risk/monitoring.py`. Fix the bin edges from development, handle empty bins, and include missing values as a bin. Tests: identical distributions → 0; swapping arguments gives the same value (symmetry); a known hand-computed case.
2. **Reference distributions:** score deciles of `dev_train` (edges saved), and scorecard bins per characteristic.
3. **Monthly score PSI** for every month 2015-01 → 2018-12 (OOT + monitor), with a line chart and 0.10 / 0.25 reference lines.
4. **Monthly CSI heatmap:** characteristics × months. Which characteristics drive any score drift? Add the points-shift view for the top movers.
5. **Investigate** the biggest real shift: is it borrower behaviour, a Lending Club policy or product change, or a data artefact (e.g. a field's definition or availability changing)? Write 3–5 sentences.
6. **Trigger rules in `config.py`:** thresholds, persistence, and the action for each RAG combination (e.g. Amber → investigate; Red on score PSI for 2 consecutive months **or** ≥ 2 key characteristics Red → recalibrate or redevelop review).
7. **Synthetic shock:** copy one monitor month, apply a clearly documented shock (e.g. raise DTI by 25% for a random 40% of applicants, or over-sample the riskiest segment), re-score it, and show that PSI/CSI and the trigger fire. Label it **SYNTHETIC** everywhere.
8. **`scripts/monitor.py --month YYYY-MM`** writes `reports/monitoring/YYYY-MM.md`: volumes, score summary, PSI (RAG), top CSI movers (RAG), approval rate at the cutoff, trigger decision.
9. Update PROGRESS.md (max PSI, first real trigger month if any, shock detected).
10. *Stretch:* **performance monitoring** by quarterly vintage where outcomes are mature (OOT and early 2016), covering Gini and calibration-in-the-large, with a written censoring caveat.

## 5. Acceptance criteria

- [ ] `uv run pytest tests/test_monitoring.py` passes (zero-shift, symmetry, hand-computed case, empty-bin case)
- [ ] Bin edges come from development and are stored; they are never recomputed on monitoring data
- [ ] Monthly PSI chart and CSI heatmap saved to `reports/figures/`
- [ ] Trigger rules live in config; the notebook and script both read them from there
- [ ] Synthetic shock fires the trigger, and the report clearly marks it as synthetic
- [ ] `scripts/monitor.py --month 2017-06` produces a readable report
- [ ] Written interpretation of the largest real drift
- [ ] Notebook passes Restart & Run All

## 6. Hints

### Task 1: PSI

<details><summary>Hint 1 (nudge)</summary>

What happens to ln(actual/expected) when a bin is empty in one of the two samples? Decide on a fix before you write the loop.
</details>

<details><summary>Hint 2 (approach)</summary>

Compute bin edges once from the reference sample (quantiles of dev scores). Use open-ended outer edges (−∞, +∞) so out-of-range future values still land in a bin. Count the share per bin in both samples, floor tiny shares with a small epsilon, then apply the formula. Return the per-bin contributions too, because they're useful in the report.
</details>

<details><summary>Hint 3 (near-solution)</summary>

```
edges = quantiles(reference, 0..1 step 0.1); edges[0] = -inf; edges[-1] = +inf   # store these
def psi(expected_values, actual_values, edges, eps=1e-4):
    e = share per bin of expected_values (+ a missing-value bin)
    a = share per bin of actual_values   (+ a missing-value bin)
    e, a = clip(e, eps), clip(a, eps)
    contrib = (a - e) * ln(a / e)
    return contrib.sum(), contrib
```
</details>

### Task 7: a meaningful synthetic shock

<details><summary>Hint 1 (nudge)</summary>

The shock should resemble something that *could* happen to a lender: a new marketing channel, an economic downturn, a policy change. Which characteristics would that move?
</details>

<details><summary>Hint 2 (approach)</summary>

Use a seeded random generator. Apply the shock to a copy, never to the stored data. Run it through the *same* scoring and monitoring functions as real months. Try two sizes: a small one that should stay Green or Amber, and a large one that should go Red. That shows your thresholds are calibrated, not just that "something fires".
</details>

## 7. Common pitfalls

- **Recomputing bins on the new data.** Then you're measuring nothing. Edges are frozen at development.
- **Empty bins → infinity or NaN.** Handle them explicitly.
- **Small monthly samples** produce noisy PSI. Check the counts; consider quarterly windows for thin segments.
- **Treating 0.1/0.25 as laws.** They're conventions. Explain your choice and the persistence rule.
- **Monitoring only the score.** A stable score can hide offsetting shifts in two characteristics.
- **Using immature outcomes as if final.** 2017 "bad rates" are mostly censoring.
- **Monitoring the approved population only.** In real life you'd monitor all applicants. Here the data limits you, so say so.

## 8. Self-check questions

1. Write the PSI formula and explain each term. Why is PSI symmetric in expected and actual?
2. Score PSI is 0.05 but CSI for `dti` is 0.30. What's going on, and is it a problem?
3. Why can't a bank simply monitor AUC every month instead of PSI?
4. Distinguish recalibration, retraining and redevelopment. Which fits a pure shift in the average bad rate with stable rank-ordering?
5. Name two weaknesses of PSI as a drift measure.
