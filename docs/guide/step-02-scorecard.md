# Step 2: Classic Scorecard (WoE/IV, Logistic Regression, Points)

**Time:** 7 h core + 2 h stretch · **Milestone:** M2 · **Notebook:** `03_scorecard` · **Modules:** `binning.py`, `scorecard.py`, `metrics.py` · **Requirements:** FR-2.1–2.8

---

## 1. Goal

Build a points-based application scorecard:
- binned characteristics, weight-of-evidence encoding and IV-based selection,
- a logistic regression on WoE,
- a points table scaled with PDO / base score / base odds,
- AUC, Gini and KS on every sample, with a gains table.

## 2. Why it matters in bank credit risk

The WoE logistic scorecard is the champion model in most retail banks:
- Every attribute's points can be read, challenged and signed off.
- Monotonic bins match business intuition.
- Decline reasons fall straight out of the points table.
- Stability is easy to monitor per characteristic.

When you interview for a credit risk role, this is the model they expect you to explain on a whiteboard.

## 3. Key concepts

| Concept | In one breath |
|---|---|
| **Fine vs coarse classing** | Start with many small bins (e.g. 20 quantiles), then merge them into a few bins that are monotonic, large enough and make business sense |
| **WoE** | For a bin: ln(share of all goods in the bin ÷ share of all bads in the bin). Positive = safer than average (in this convention). It puts every variable on the same log-odds scale |
| **IV** | Σ over bins of (good share − bad share) × WoE. One number for a variable's separating power. Rough guide: < 0.02 useless, 0.02–0.1 weak, 0.1–0.3 medium, > 0.3 strong, > 0.5 *suspicious* |
| **Monotonicity** | Bad rate should move one way across ordered bins (e.g. higher DTI → riskier). Zig-zags are usually noise, and validators will ask about them |
| **Logistic regression on WoE** | Each coefficient scales a variable's WoE. With WoE = ln(G/B) and target = bad, all coefficients should share one sign |
| **Points scaling** | Score = Offset + Factor × ln(odds of good), with Factor = PDO / ln 2 and Offset = BaseScore − Factor × ln(BaseOdds). PDO = points to double the odds |
| **AUC / Gini / KS** | AUC = probability a random good outscores a random bad; Gini = 2·AUC − 1; KS = maximum gap between the good and bad cumulative score distributions |
| **Gains table** | Score bands (e.g. deciles) with counts, bad rate and cumulative % of bads captured. It's what business users actually read |

**Resources**
- Siddiqi, N. (2017). *Intelligent Credit Scoring* (2nd ed.), Wiley. Read the chapters on scorecard development and scaling.
- optbinning documentation (`OptimalBinning`, `BinningProcess`, `Scorecard`): <https://gnpalencia.org/optbinning/>

## 4. Tasks (in order)

1. **By hand first:** compute the WoE/IV table for one numeric variable (e.g. `dti`, 10 quantile bins) and one categorical (e.g. `home_ownership`) on `dev_train` with plain pandas. Then reproduce them with optbinning and reconcile any differences (sign convention? epsilon?).
2. **Bin all candidates** on `dev_train` only. Choose: monotonic trend where business logic says so, minimum bin size (e.g. ≥ 5% of rows), missing values as their own bin, special values (e.g. the "never delinquent" meaning) handled deliberately.
3. **IV table** → `reports/iv_table.csv`. Drop IV < 0.02. For IV > 0.5, go back to the Step 1 audit before keeping it.
4. **Review bins visually** (bad rate per bin with counts). Adjust bins that make no business sense, and log each manual change.
5. **Correlation among WoE features** (and optionally VIF). From each highly correlated pair (|r| > ~0.7), keep the more predictive or more explainable one.
6. **Fit logistic regression** on WoE features with `statsmodels` (you need p-values and clean coefficients). Check signs and significance. Remove offenders one at a time and refit. Target: **8–15 characteristics**.
7. **Scale to points.** Choose PDO, base score and base odds (common: 600 points at 50:1 good:bad, PDO 20), and record them in `config.py`. Produce `reports/scorecard_table.csv`: characteristic, bin, WoE, points.
8. **Evaluate** on dev_train, dev_valid and OOT: AUC, Gini, KS, plus a gains table by score decile. Compare with `sub_grade` alone as a benchmark ranker.
9. **Code quality:** put AUC/Gini/KS/gains in `src/credit_risk/metrics.py`, with tests on a tiny hand-made example where you know the answer. Put scaling in `scorecard.py`, with a test of the "PDO doubles the odds" property.
10. Save the binning, model and metrics (`models/`, `reports/metrics/scorecard.json`), then update PROGRESS.md.
11. *Stretch:* preview stability by comparing each characteristic's bin distribution in dev_train vs OOT (the CSI you'll formalise in Step 6).

## 5. Acceptance criteria

- [ ] Hand-computed WoE/IV matches the library to within rounding, and you can explain any convention difference
- [ ] All binning was fit on `dev_train` only (assert or comment where it's fit)
- [ ] Every final characteristic has IV ≥ 0.02, sensible (mostly monotonic) bins and a consistent coefficient sign
- [ ] Each removed variable has a logged reason
- [ ] Scorecard table saved; summing an applicant's points reproduces their score from the model (spot-check 3 rows)
- [ ] Metrics table for 3 samples + `sub_grade` benchmark recorded in PROGRESS.md
- [ ] `uv run pytest tests/test_metrics.py tests/test_scorecard_scaling.py` passes
- [ ] Notebook passes Restart & Run All

## 6. Hints

### Task 1: WoE/IV by hand

<details><summary>Hint 1 (nudge)</summary>

You need, per bin: the count of goods, the count of bads, and each as a share of the *column total* of goods or bads. Everything else follows.
</details>

<details><summary>Hint 2 (approach)</summary>

Group by bin and aggregate the target into counts of 1s and 0s. Divide each by its total. WoE is the log of the ratio; IV sums (difference × WoE). Add a tiny epsilon or merge bins to avoid log(0). Then check: does WoE increase or decrease as bad rate rises? That tells you the convention.
</details>

<details><summary>Hint 3 (near-solution)</summary>

```
t = group dev_train by bin: bads = sum(target), goods = count - bads
t.good_share = goods / goods.sum();  t.bad_share = bads / bads.sum()
t.woe = ln((good_share + eps) / (bad_share + eps))
t.iv_part = (good_share - bad_share) * woe
IV = t.iv_part.sum()
```
</details>

### Task 6: coefficient signs

<details><summary>Hint 1 (nudge)</summary>

If every variable's WoE points the same way (higher = safer), what sign should every coefficient have when you predict *bad*?
</details>

<details><summary>Hint 2 (approach)</summary>

A flipped sign usually means multicollinearity: two variables carrying the same information, so the model uses one to "correct" the other. Look at the correlation of the offender with the others, remove one of the pair, and refit. Don't silently keep a flipped sign; a validator *will* ask.
</details>

### Task 7: points scaling

<details><summary>Hint 1 (nudge)</summary>

Write down what you want: "score S0 at odds O0, and every +PDO points doubles the odds". That gives two equations in two unknowns (Factor, Offset).
</details>

<details><summary>Hint 2 (approach)</summary>

The model's linear predictor is the log-odds of *bad*. The scorecard uses log-odds of *good*, which is its negative. Distribute the intercept evenly across the n characteristics so each attribute gets whole-number points that add up to the score.
</details>

<details><summary>Hint 3 (near-solution)</summary>

```
factor = PDO / ln(2)
offset = BASE_SCORE - factor * ln(BASE_ODDS)
for each characteristic i with coefficient b_i, for each bin j with woe_ij:
    points_ij = -(b_i * woe_ij + intercept / n) * factor + offset / n
score(applicant) = sum of points of the bins they fall into
# test: two applicants whose model odds differ by 2x should differ by PDO points
```
</details>

## 7. Common pitfalls

- **Binning on all data** (including OOT). The bins have then "seen the future".
- **Bins with zero bads or zero goods** → infinite WoE. Merge them or use minimum bin size constraints.
- **Too many bins.** It overfits, and nobody can sign off on 14 bins for `annual_inc`.
- **Mixing WoE sign conventions** (ln(G/B) vs ln(B/G)) between your code and the library, so coefficient signs "look wrong".
- **scikit-learn `LogisticRegression` defaults** apply L2 regularisation (`C=1.0`) and give no p-values. Know what you're fitting.
- **Reporting dev_valid metrics as final** after you used dev_valid for selection. The final number is OOT.
- **Forgetting the missing bin.** At scoring time, a missing value must get points too.

## 8. Self-check questions

1. Explain WoE and IV to a non-technical credit manager in two sentences each.
2. Why bin continuous variables rather than feeding raw values to logistic regression? What do you lose?
3. A variable has IV = 0.65. What are your first three checks?
4. With PDO = 20, an applicant's score rises from 600 to 640. What happens to their odds of being good?
5. One coefficient has the opposite sign to all the others. What does it indicate, and what do you do?
