# Week 0 — Daily Plan (2 days)

**Goal for the week:** a working, installable package that imports, passes one trivial test, and is pushed to GitHub. **No statistics yet** — this week is pure engineering scaffolding.

**Dates:** Day 1 = Saturday Aug 22, 2026 · Day 2 = Sunday Aug 23, 2026
**Estimated total:** 4–6 hours · **You chose:** 2 days, ~3 hours each.

---

## Current repo state (verified before writing this plan)

Everything below already exists — you are further along than the build guide assumes:

- `pyproject.toml` (uv scaffold) — but `description` is a placeholder and `dependencies = []`
- `src/abtest_toolkit/__init__.py` — only the default `Hello from abtest-toolkit!` function
- `.venv`, `.python-version` (3.14), `.gitignore`
- Git repo initialized but **zero commits**
- **Missing:** `tests/`, all module files, `simulations/`, `case_study/`, any dependencies

### Two things to know before you start

1. **Package name.** The build guide says `src/abtest/`, but uv scaffolded `src/abtest_toolkit/`. Keep the `abtest_toolkit` name — it's the import name. All modules live in `src/abtest_toolkit/`.
2. **`.venv` already exists**, which means uv is almost certainly installed. If `uv --version` works, skip the install step.

---

## Day 1 — Tooling, dependencies, skeleton (~3h)

### Session 1 — Verify/install tooling (45 min)

Open PowerShell in the project folder:

```powershell
cd C:\Users\NITRO\Desktop\Projects\abtest-toolkit
```

1. **uv**
   ```powershell
   uv --version
   ```
   If it prints a version, skip install. If "not recognized", install:
   ```powershell
   irm https://astral.sh/uv/install.ps1 | iex
   ```
   Then **close and reopen the terminal** so the PATH updates, and re-run `uv --version`.

2. **Git**
   ```powershell
   git --version
   ```
   If missing, install Git for Windows from https://git-scm.com and reopen the terminal.

3. **Git identity** (one-time, if you haven't done it):
   ```powershell
   git config --global user.name "AHMED REDA ADDA ABBOU"
   git config --global user.email "ahmed.addaabbou@gmail.com"
   ```

4. **GitHub:** make sure you can log in at github.com. (You can push over HTTPS; GitHub will ask for a Personal Access Token when you push.)

**Done when:** `uv --version` and `git --version` both print versions.

### Session 2 — Add dependencies (30 min)

```powershell
uv add numpy scipy pandas matplotlib
uv add --dev pytest pytest-cov ruff statsmodels jupyter
```

Two notes from the guide:

- `statsmodels` is a **dev** dependency on purpose — you'll use it in tests to cross-check your own implementations, but the package itself must not depend on it.
- Run `uv add` *after* `cd` into the project so it edits the right `pyproject.toml`.

**Then fix `pyproject.toml`** — change the placeholder line:

```toml
description = "Add your description here"
```

to:

```toml
description = "A/B testing toolkit: design, diagnostics, analysis, and variance reduction."
```

Verify the deps landed:

```powershell
uv sync
```

**Done when:** `uv sync` runs without errors and `pyproject.toml` lists the dependencies.

### Session 3 — Build the skeleton (1h)

Create the folders and empty module files:

```powershell
mkdir tests simulations case_study
```

Create these files under `src/abtest_toolkit/` — one per area of the package:

- `design.py` — sample size, MDE, experiment duration
- `diagnostics.py` — SRM check, A/A test, winsorize
- `analysis.py` — Welch t-test, z-test, Mann-Whitney, bootstrap
- `variance.py` — delta method, CUPED
- `sequential.py` — O'Brien-Fleming boundaries
- `report.py` — `ExperimentResult` output type

Each empty file gets a one-line docstring stub, e.g.:

```python
"""Sample-size design and experiment planning."""
```

Replace the contents of `src/abtest_toolkit/__init__.py` with the package version:

```python
"""A/B testing toolkit: design, diagnostics, analysis, and variance reduction."""

__version__ = "0.1.0"
```

(You can delete the old `main()` function — there's no CLI yet. Also remove or keep the `[project.scripts]` entry in `pyproject.toml`; either is fine this week.)

**Done when:** the module structure looks like:

```
src/abtest_toolkit/
  __init__.py
  design.py
  diagnostics.py
  analysis.py
  variance.py
  sequential.py
  report.py
```

### Session 4 — Head start on the first test (45 min, optional)

If you still have steam, do the Day 2 Session 1 work now (it's only ~30 minutes). Otherwise stop here and pick up tomorrow.

**End of Day 1 checkpoint:** `uv sync` succeeds, dependencies installed, six module stubs created, `__version__` set.

---

## Day 2 — First test, git, GitHub (~3h)

### Session 1 — First passing test (1h)

Put a deliberately trivial function in `src/abtest_toolkit/design.py`:

```python
def hello() -> str:
    return "abtest-toolkit"
```

Create `tests/test_design.py`:

```python
from abtest_toolkit.design import hello


def test_hello():
    assert hello() == "abtest-toolkit"
```

Run the suite:

```powershell
uv run pytest
```

**Milestone for the week — you should see `1 passed`.**

### Session 2 — Git and GitHub (1h)

The repo is already initialized, so no `git init`. Make the first commit:

```powershell
git add .
git commit -m "Initial package skeleton"
```

Create an **empty** repo named `abtest-toolkit` on GitHub (no README, no .gitignore — you already have both), then:

```powershell
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/abtest-toolkit.git
git push -u origin main
```

**Commit discipline from here on** (the guide is emphatic): one commit per function or meaningful change, message explains *what changed and why*. `Add delta method for ratio metric variance` — not `update`. Recruiters read your commit history.

The week's DoD requires **at least 3 commits on GitHub**. Suggested for Day 2:

1. `Initial package skeleton` (done above)
2. `Add hello stub and first test`
3. `Add dependency metadata and module stubs` (or any real Week 1 head-start commit)

### Session 3 — Verify the definition of done (45 min)

Run all three Week 0 checks:

- [ ] `uv run pytest` — passes
- [ ] Repo is on GitHub with **at least 3 commits**
- [ ] `uv run python -c "import abtest_toolkit; print(abtest_toolkit.__version__)"` — prints `0.1.0`

**Then prep Week 1 (optional but recommended):** re-read the Week 1 section of `abtest-toolkit-build-guide.md`. Look at `sample_size_proportions` and `check_srm` — those are your next two functions. Skim the formula in Step 1.1 so the math is warm tomorrow.

---

## Week 0 — Definition of done

- [ ] `uv run pytest` passes
- [ ] Repo on GitHub with at least 3 commits
- [ ] `uv run python -c "import abtest_toolkit; print(abtest_toolkit.__version__)"` → `0.1.0`

---

## Stuck? (from the build guide)

**Import errors.** Almost always the layout. Confirm `src/abtest_toolkit/__init__.py` exists and run `uv sync` again.

**`uv` not found after install.** Close and reopen the terminal — PATH updates only apply to new shells.

**Push asks for credentials.** Use a GitHub Personal Access Token as your password (Settings → Developer settings → Personal access tokens), or set up SSH.

**A command errors with a path problem.** You're probably not in the project directory — `cd` into `C:\Users\NITRO\Desktop\Projects\abtest-toolkit` first.

---

## Notes for this week's sessions (from the guide)

- Week 0 feels disproportionately hard because it's all new engineering, not math you already know. That's normal and it passes.
- `uv` handles Python versions, virtual environments, and dependencies in one tool — the `.venv` is already created; always use `uv run` instead of activating the venv manually.
