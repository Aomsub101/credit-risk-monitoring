# Step 5: Business Layer (Profit-Based Cutoff)

**Time:** 3 h core + 1.5 h stretch · **Milestone:** M4 (with Step 6) · **Notebook:** `06_business_cutoff` · **Module:** `business.py` · **Requirements:** FR-5.1–5.6

---

## 1. Goal

Translate calibrated PDs into a **decision rule**:
- one score cutoff, chosen from an expected-profit curve and a risk appetite,
- with the trade-off between approval rate and bad rate made explicit,
- and its sensitivity to assumptions shown.

## 2. Why it matters in bank credit risk

A model doesn't make money; a **strategy** does. Credit committees approve cutoffs, not AUCs. They want to see:
- the approval rate vs bad rate curve,
- what the expected profit is at each candidate cutoff,
- how wrong the assumptions can be before the decision changes.

Being able to talk about this is what separates a credit risk modeller from a generic data scientist.

## 3. Key concepts

| Concept | In one breath |
|---|---|
| **Expected profit per applicant** | (1 − PD) × revenue-if-good − PD × loss-if-bad (− fixed costs). Approve if it's positive |
| **EAD / LGD** | Exposure at default (how much is owed when it goes bad) and loss given default (share not recovered). Loss-if-bad ≈ EAD × LGD |
| **Breakeven PD** | The PD where expected profit = 0. With revenue G and loss L: PD* = G / (G + L) |
| **Strategy curve** | For every cutoff: approval rate on the x-axis, bad rate among approved on the y-axis. Moving the cutoff slides along it |
| **Risk appetite** | A constraint set by management (e.g. "bad rate among approved ≤ X%"). It may force a cutoff stricter than the profit maximum |
| **Sensitivity analysis** | Re-run the cutoff choice with LGD ± 20%, revenue ± 20%. A robust cutoff barely moves |
| **Swap set** | Applicants approved by one model but declined by the other at the same approval rate. Committees look at who gets swapped in and out |
| **Accepted-only caveat** | Every applicant here was already approved by Lending Club, so "approval rate" means *approval on top of LC's policy*, not of all applicants |

**Resources**
- Baesens, B., Rösch, D. & Scheule, H. (2016). *Credit Risk Analytics*, Wiley. Read the chapters on scorecard use and strategy.
- Verbraken, T., Bravo, C., Weber, R. & Baesens, B. (2014). "Development and application of consumer credit scoring models using profit-based classification measures." *European Journal of Operational Research* 238(2).

## 4. Tasks (in order)

1. **Write the assumptions first** (markdown cell + PROGRESS.md decision log): revenue if good (e.g. interest over the expected life, net of cost of funds), loss if bad (EAD × LGD), and any per-loan cost. Keep it simple and explicit. Every number needs a one-line justification.
2. **Per-applicant expected profit** on `dev_valid` and OOT using **calibrated** PDs and loan amount / interest rate. (Using `int_rate` here is fine even if it's not a model feature. Why?)
3. **Strategy table:** for a grid of score cutoffs (e.g. every 5 points, or every percentile), show approval rate, bad rate among approved, number of bads avoided, total and per-application expected profit.
4. **Plots:** expected profit vs cutoff; strategy curve (approval rate vs bad rate) with the chosen point marked.
5. **Choose the cutoff on `dev_valid`:** profit-maximising, then check against a risk appetite you set (e.g. max bad rate). Report its performance on **OOT**.
6. **Sensitivity:** recompute the optimal cutoff for LGD and revenue at ±20% (a small grid). Show how far the cutoff moves.
7. Save the cutoff to `models/cutoff.json` (score, PD, assumptions, date). Put the logic in `src/credit_risk/business.py`.
8. Update PROGRESS.md (cutoff, approval rate, bad rate, profit per application).
9. *Stretch:* **realised-profit back-test** on OOT using actual repayments (`total_pymnt`, `recoveries`, `funded_amnt`). Compare it with expected profit. These columns are leakage as *features* but legitimate as *outcomes*.
10. *Stretch:* **swap-set analysis** between scorecard and LightGBM at the same approval rate. Who's swapped, and what's their bad rate?

## 5. Acceptance criteria

- [ ] Assumptions table with justifications exists in the notebook and the decision log
- [ ] Profit uses calibrated PDs (state which calibrator)
- [ ] Strategy table + 2 plots saved to `reports/`
- [ ] Cutoff chosen on `dev_valid`, reported on OOT: approval rate, bad rate, profit per application
- [ ] Sensitivity grid shows the cutoff range under ±20% assumption shocks
- [ ] `models/cutoff.json` written; `business.py` has the profit and strategy-table functions
- [ ] Notebook passes Restart & Run All

## 6. Hints

### Task 1: a simple, defensible profit model

<details><summary>Hint 1 (nudge)</summary>

For a 36-month amortising loan that's fully repaid, is the total interest earned `loan_amnt × int_rate × 3`? Why not?
</details>

<details><summary>Hint 2 (approach)</summary>

Amortising balances shrink, and many borrowers prepay, so average exposure over the life is well below the original amount. Use a simplifying factor for average outstanding balance and state it. For bads, assume a portion is repaid before default, then apply LGD to the remaining exposure. The point isn't precision, it's **explicit, checkable assumptions**. Your stretch back-test will tell you how far off they are.
</details>

<details><summary>Hint 3 (near-solution)</summary>

```
revenue_if_good = loan_amnt * (int_rate - cost_of_funds) * term_years * avg_balance_factor
loss_if_bad     = loan_amnt * ead_factor * LGD
exp_profit_i    = (1 - pd_i) * revenue_if_good_i - pd_i * loss_if_bad_i
for cutoff in grid:
    approved = score >= cutoff
    row = (cutoff, approved.mean(), y[approved].mean(), exp_profit[approved].sum() / n_applicants)
```
</details>

### Task 6: sensitivity

<details><summary>Hint 1 (nudge)</summary>

Wrap "choose optimal cutoff given assumptions" in one function. Then sensitivity is just calling it in a loop.
</details>

## 7. Common pitfalls

- **Using uncalibrated scores as PDs.** The profit maths assumes real probabilities.
- **Choosing the cutoff on OOT and reporting OOT profit as unbiased.**
- **Forgetting the accepted-only caveat** when you talk about approval rates.
- **Hiding assumptions in code.** A committee can't challenge a number it can't see.
- **One-point answers.** A cutoff with no sensitivity range invites the question "what if LGD is wrong?"
- **Profit without cost of funds or operating cost.** It overstates how good marginal applicants are.

## 8. Self-check questions

1. Derive the breakeven PD when a good loan earns G and a bad loan loses L.
2. Why might a bank choose a cutoff stricter than the profit maximum? And why might it choose a looser one?
3. Your data contains only loans Lending Club approved. What does that mean for the approval rates you report?
4. If LGD rises from 60% to 80%, which way does the optimal cutoff move, and why?
5. What is a swap set, and why do credit committees ask for it when changing models?
