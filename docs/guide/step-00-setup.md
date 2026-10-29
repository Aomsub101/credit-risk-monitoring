# Step 0: Setup

**Time:** ~1.5 h · **Milestone:** M0 · **Requirements:** TR-1, TR-3, TR-5, DR-1, DR-4

> How to use these guides: work through the **Tasks** in order. When stuck, open **Hint 1** first. Open the next hint only if you're still stuck after ~15 minutes. Answer the **self-check questions** in `docs/PROGRESS.md`.

---

## 1. Goal

A reproducible environment, the raw data in place with a recorded checksum, paths that work from any folder, and your first commits in the new convention.

## 2. Why it matters in bank credit risk

A model validator's first request is usually *"rerun it"*. If your results depend on an unpinned library version or on which folder Jupyter started in, the model fails validation before anyone looks at its Gini. Reproducibility is a model-risk requirement, not a nice-to-have.

## 3. Key concepts

| Concept | In one breath |
|---|---|
| **Lockfile** (`uv.lock`) | Records the *exact* version of every package (including dependencies of dependencies). `pyproject.toml` says "pandas ≥ 3.1"; the lockfile says "pandas 3.1.0, numpy 2.5.4, …" |
| **src layout** | Your importable package lives in `src/credit_risk/`. `uv sync` installs it into `.venv` in editable mode, so `import credit_risk` works from notebooks, scripts and tests without `sys.path` hacks |
| **Project-root paths** | Build every path from the location of a known file (e.g. `config.py`), never from the current working directory |
| **Checksum** | A SHA-256 hash proves that the file you trained on is byte-for-byte the file someone else downloads |

**Resources**
- uv docs, "Working on projects": <https://docs.astral.sh/uv/guides/projects/>
- [docs/COMMIT_CONVENTION.md](../COMMIT_CONVENTION.md)

## 4. Tasks

1. **Clean up space (optional).** `data/home-credit-default-risk.zip` (~720 MB) is still on disk. Delete it if you need the space; it's gitignored either way.
2. **Download the data** into `data/raw/lending-club/` (already created):
   - Kaggle → [All Lending Club loan data](https://www.kaggle.com/datasets/wordsforthewise/lending-club) → download **only** `accepted_2007_to_2018Q4.csv.gz` (~392 MB). Skip the `rejected_…` file. **Keep it compressed**, because pandas reads `.gz` directly.
   - Data dictionary: `LCDataDictionary.xlsx` from Kaggle [Lending Club Loan Data](https://www.kaggle.com/datasets/wendykan/lending-club-loan-data) (download just that file) or the [figshare copy](https://figshare.com/articles/dataset/Lending_club_dataset_description/20016077).
   - Record the SHA-256 and download date in `docs/PROGRESS.md` → *Data provenance*:
     `Get-FileHash data\raw\lending-club\accepted_2007_to_2018Q4.csv.gz -Algorithm SHA256`
3. **Create the uv project** (run in the repo root; tested on your setup, and it keeps your README):
   ```powershell
   uv init --lib --name credit-risk --python 3.12
   uv add pandas numpy pyarrow scikit-learn statsmodels lightgbm optbinning shap matplotlib seaborn streamlit joblib
   uv add --dev pytest ruff ipykernel
   ```
   Then edit `pyproject.toml` → `description`, and replace the placeholder `hello()` in `src/credit_risk/__init__.py` with a one-line docstring.
4. **Extend `.gitignore`.** It currently ignores only `/data`. Add `models/`, the virtual environment, and the Python/Jupyter/pytest/ruff cache folders.
5. **Create `src/credit_risk/config.py`** with: `PROJECT_ROOT`, `RAW_FILE`, `INTERIM_DIR`, `PROCESSED_DIR`, `MODELS_DIR`, `REPORTS_DIR`, and `SEED = 42`. Use `pathlib` only.
6. **Connect VS Code.** Open `notebooks/01_extract_data.ipynb` → *Select Kernel* → the `.venv` interpreter in this repo.
7. **Verify** (see acceptance criteria).
8. **Commit** in logical pieces following [COMMIT_CONVENTION.md §7](../COMMIT_CONVENTION.md#7-your-first-commits-right-now), then one more `build:` commit for `pyproject.toml` + `uv.lock` + `.python-version`, and a `chore:` commit for `.gitignore`. Push to `dev`.

## 5. Acceptance criteria

- [ ] `uv sync --locked` completes with no changes
- [ ] This prints `True`:
  `uv run python -c "import lightgbm, optbinning, shap, streamlit; from credit_risk.config import RAW_FILE; print(RAW_FILE.exists())"`
- [ ] The same `RAW_FILE.exists()` check returns `True` from inside a notebook in `notebooks/`
- [ ] `git status` does not list `.venv/`, `data/` or cache folders
- [ ] SHA-256 recorded in PROGRESS.md
- [ ] `git log --oneline` shows conventional-commit messages

## 6. Hints

### Project-root paths in `config.py`

<details><summary>Hint 1 (nudge)</summary>

`config.py` always lives at the same place relative to the repo root, no matter where Python was started. What does `__file__` give you?
</details>

<details><summary>Hint 2 (approach)</summary>

Resolve `__file__` to an absolute `Path`, walk up the right number of parents to reach the repo root, then join the sub-paths with `/`. Count the levels: `config.py` → `credit_risk/` → `src/` → root.
</details>

<details><summary>Hint 3 (near-solution)</summary>

```
PROJECT_ROOT = absolute path of this file, then up N parents   # N = number of folders between it and the repo root
DATA_DIR     = PROJECT_ROOT / "data"
RAW_FILE     = DATA_DIR / "raw" / "lending-club" / "<file name>"
... same pattern for interim, processed, models, reports
SEED = 42
```
Check it: `print(PROJECT_ROOT)` from a notebook and from a terminal, which should give the same output.
</details>

### `.gitignore`

<details><summary>Hint 1 (nudge)</summary>

Run `git status` after `uv sync`. Anything listed that you would never want on GitHub belongs in `.gitignore`.
</details>

<details><summary>Hint 2 (approach)</summary>

You need patterns for: the model folder, the virtual environment folder, Python bytecode caches, Jupyter checkpoints, and the pytest and ruff caches. GitHub's `Python.gitignore` template is a good reference, but only take what you need.
</details>

## 7. Common pitfalls

- Opening the CSV in Excel: Excel silently truncates at 1,048,576 rows. LC has about 2.26M.
- Unzipping the `.gz` "to make it faster". It costs ~1.6 GB of disk and gains little. Read it once, then work from parquet.
- Running `pip install` inside the venv instead of `uv add`. The lockfile won't know about the package.
- Forgetting to commit `uv.lock`. Without it, "pinned dependencies" isn't true.
- Using `r"data\..."` paths. Backslash paths break on Linux (CI, Streamlit Cloud).

## 8. Self-check questions

1. What's the difference between `pyproject.toml` and `uv.lock`, and why commit both?
2. Why does `os.getcwd()` give different answers in a notebook vs a terminal?
3. What does `uv sync --locked` do that `uv sync` doesn't?
4. Why record a SHA-256 for a public dataset anyone can download?
