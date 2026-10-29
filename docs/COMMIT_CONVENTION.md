# Commit Convention

This repo follows **[Conventional Commits 1.0.0](https://www.conventionalcommits.org/en/v1.0.0/)**, the most widely used standard. A reader (or a tool) can tell from the first word what a commit did.

## 1. Format

```
<type>(<optional scope>): <subject>
                                          ← blank line
<optional body: what and WHY, wrapped at 72 chars>
                                          ← blank line
<optional footer(s): Refs: #12, BREAKING CHANGE: ...>
```

Example:

```
fix(data): exclude post-origination payment columns from features

total_pymnt and recoveries are only known after the loan is issued.
Including them pushed OOT AUC to 0.97, a classic leakage signature.
Added both to LEAKAGE_COLUMNS and a test that guards the feature list.

Refs: FR-1.2
```

## 2. Subject line rules

| Rule                                                | Good                                          | Bad                                     |
| --------------------------------------------------- | --------------------------------------------- | --------------------------------------- |
| Imperative mood ("add", not "added")                | `add vintage bad-rate chart`                | `added vintage chart`                 |
| Lowercase after the colon                           | `feat(eda): add missingness heatmap`        | `feat(eda): Add Missingness Heatmap`  |
| No trailing period                                  | `… heatmap`                                | `… heatmap.`                         |
| ≤ 72 characters (aim for ~50)                      |                                               | a full paragraph in the subject         |
| Says **what changed**, not what you did today | `feat(scorecard): scale log-odds to points` | `update notebook`, `wip`, `stuff` |

**Test:** finish the sentence *"If applied, this commit will ___."* Your subject should fit the blank.

## 3. Types

Conventional Commits comes from software engineering. Here is how each type maps to data science work in this project:

| Type         | Use it when you…                                                  | Example from this project                                       |
| ------------ | ------------------------------------------------------------------ | --------------------------------------------------------------- |
| `feat`     | add a new capability, analysis or result                           | `feat(scorecard): add WoE binning with monotonic constraints` |
| `fix`      | correct a bug **or a wrong result** (including leakage)      | `fix(split): stop OOT rows leaking into dev_valid`            |
| `refactor` | restructure code with **no change in output**                | `refactor(data): move target mapping from notebook to src`    |
| `perf`     | make something faster or lighter, same output                      | `perf(data): read raw file with usecols and category dtypes`  |
| `test`     | add or fix tests only                                              | `test(monitoring): add PSI symmetry and zero-shift tests`     |
| `docs`     | change documentation only                                          | `docs(progress): log step 1 decisions and metrics`            |
| `build`    | change dependencies or packaging (`pyproject.toml`, `uv.lock`) | `build: add lightgbm and shap`                                |
| `ci`       | change CI config (GitHub Actions)                                  | `ci: run ruff and pytest on push`                             |
| `style`    | formatting only (ruff format, whitespace)                          | `style: apply ruff format`                                    |
| `chore`    | housekeeping that fits nothing above (moves, `.gitignore`)       | `chore: ignore models/ and cache folders`                     |
| `revert`   | undo a previous commit                                             | `revert: feat(scorecard): add int_rate feature`               |

**Breaking change:** add `!` after the type/scope (`feat(config)!: rename DEV_END to TRAIN_END`) and explain in a `BREAKING CHANGE:` footer. This is rare in a solo project, but it's good to know.

## 4. Scopes for this repo

Use the part of the project you touched. Keep this list short and consistent:

`env` · `data` · `eda` · `split` · `features` · `binning` · `scorecard` · `lgbm` · `metrics` · `calibration` · `explain` · `business` · `monitoring` · `pipeline` · `app` · `readme` · `guide` · `progress`

Leave the scope out when a change spans many areas (e.g. `chore: move notebook into notebooks/`).

## 5. Habits for data science repos

1. **One logical change per commit.** "Add column audit" and "fix dtype bug" are two commits. Stage selectively with `git add <path>`.
2. **Restart & Run All before committing a notebook.** Stale outputs (like the one in your old cell 3) mislead readers. Outputs are fine to keep in a portfolio repo, because GitHub renders them, but trim huge outputs: no 1,000-row DataFrame prints.
3. **Never commit** `data/`, `models/`, credentials, or anything that is not public.
4. **Commit `uv.lock` together with `pyproject.toml`** in the same `build:` commit.
5. **Reference requirements** in the footer when useful: `Refs: FR-2.5`.
6. **Don't rewrite pushed history.** Your two existing commits (`[feat] …`) stay as they are; the new convention starts from your next commit.

## 6. Branch workflow

```
main  ─────────────●─────────────────●──────  (milestones only, tagged)
                  ↑ PR              ↑ PR
dev   ──●──●──●──●──●──●──●──●──●──●──────  (integration)
         ↖ merge  ↖ merge
feat/step-01-eda    feat/step-02-scorecard     (one short-lived branch per step)
```

- Branch names: `<type>/<short-description>`, e.g. `feat/step-01-eda`, `fix/oot-leak`, `docs/model-doc`.
- At the end of each step: merge the step branch into `dev`.
- At each milestone (M1…M5): open a PR `dev` → `main`, using a conventional-commit-style title. Tag the final one `v1.0.0`.
- If per-step branches feel heavy, committing straight to `dev` is acceptable for a solo project. Keep the PR to `main` per milestone, because that's what makes your GitHub history readable.

## 7. Your first commits (right now)

The scaffold is uncommitted on `dev`. Practise the convention by committing it in logical pieces. Run `git status` after each step to see what's left.

```powershell
# 1. Repo restructure (git will detect the notebook rename)
git add "[01]_extract_data.ipynb" notebooks/ scripts/ app/ tests/ reports/
git commit -m "chore: move notebook into notebooks/ and add project folders"

# 2. Planning docs
git add docs/PROJECT_BRIEF.md docs/REQUIREMENTS.md docs/ARCHITECTURE.md docs/ROADMAP.md docs/PROGRESS.md
git commit -m "docs: add project brief, requirements, architecture and roadmap"

# 3. Guides + template + conventions
git add docs/guide/ docs/MODEL_DOCUMENTATION_TEMPLATE.md docs/COMMIT_CONVENTION.md
git commit -m "docs(guide): add step guides, model doc template and commit convention"

# 4. Mentor setup + README
git add CLAUDE.md README.md
git commit -m "docs: add mentor-mode CLAUDE.md and README skeleton"

git log --oneline -5
git push
```

> Note on commit 1: the notebook also carries your earlier edit to the exclude list. That's fine here, since the notebook will be rewritten in Step 1. You could mention it in a body line if you want to practise writing bodies: `git commit` without `-m` opens your editor.

## 8. Further reading

- [Conventional Commits 1.0.0 specification](https://www.conventionalcommits.org/en/v1.0.0/)
- Chris Beams, [How to Write a Git Commit Message](https://cbea.ms/git-commit/)
