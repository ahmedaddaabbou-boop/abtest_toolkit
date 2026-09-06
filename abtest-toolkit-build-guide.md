# Building `abtest-toolkit` — A Step-by-Step Guide

**Total time: 55–75 hours over 6 weeks** (roughly 10–12 hours/week)

A note before you start: the statistics here is the *easy* part for you. You have an MSc in applied mathematics — Welch's t-test, the delta method, and variance reduction are all things you can derive. What you haven't done before is package Python properly, write tests, and set up CI. So this guide spends more time on the engineering scaffolding than on the math, and Week 0 will feel disproportionately hard. That's normal and it passes.

---

## Table of contents

- [Week 0 — Setup](#week-0--setup-46-hours)
- [Week 1 — Design & diagnostics](#week-1--design--diagnostics-10-hours)
- [Week 2 — Analysis](#week-2--analysis-12-hours)
- [Week 3 — CUPED & the peeking simulation](#week-3--cuped--the-peeking-simulation-12-hours)
- [Week 4 — Remaining simulations & README](#week-4--remaining-simulations--readme-10-hours)
- [Week 5 — Case study & decision memo](#week-5--case-study--decision-memo-8-hours)
- [Week 6 — Sequential testing & publishing](#week-6--sequential-testing--publishing-10-hours)
- [When you get stuck](#when-you-get-stuck)
- [Resources](#resources)

---

## Week 0 — Setup (4–6 hours)

The goal this week is **not** to write statistics. It's to have an empty package that installs, imports, and passes one trivial test. If you get that far, everything after is incremental.

### Step 0.1 — Install the tooling (45 min)

Install `uv`, which handles Python versions, virtual environments, and dependencies in one tool:

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows (PowerShell)
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"
```

You also need Git configured and a GitHub account. Verify:

```bash
uv --version
git --version
```

### Step 0.2 — Create the project (30 min)

```bash
uv init abtest-toolkit --package
cd abtest-toolkit
uv add numpy scipy pandas matplotlib
uv add --dev pytest pytest-cov ruff statsmodels jupyter
```

`--package` matters — it creates a real installable package layout rather than a loose script folder.

`statsmodels` is a dev dependency, not a runtime one. You'll use it in tests to cross-check your own implementations, but your package shouldn't depend on it.

### Step 0.3 — Build the skeleton (1 hour)

Create empty module files:

```bash
mkdir -p src/abtest tests simulations case_study
touch src/abtest/{design,diagnostics,analysis,variance,sequential,report}.py
touch tests/test_design.py
```

Edit `src/abtest/__init__.py`:

```python
"""A/B testing toolkit: design, diagnostics, analysis, and variance reduction."""

__version__ = "0.1.0"
```

### Step 0.4 — Prove it works (1 hour)

Put a deliberately trivial function in `src/abtest/design.py`:

```python
def hello() -> str:
    return "abtest-toolkit"
```

And in `tests/test_design.py`:

```python
from abtest.design import hello


def test_hello():
    assert hello() == "abtest-toolkit"
```

Run it:

```bash
uv run pytest
```

When you see `1 passed`, you have a working package. This is the milestone for the week.

### Step 0.5 — Git and GitHub (1 hour)

```bash
git init
git add .
git commit -m "Initial package skeleton"
```

Create an empty repo on GitHub named `abtest-toolkit`, then:

```bash
git remote add origin https://github.com/YOUR_USERNAME/abtest-toolkit.git
git branch -M main
git push -u origin main
```

**Commit discipline from here on:** one commit per function or per meaningful change, with a message describing *what changed and why*. `Add delta method for ratio metric variance` — not `update`. Your commit history is part of what recruiters look at, and a repo with 4 commits all named "stuff" undercuts everything else.

### Week 0 definition of done

- [ ] `uv run pytest` passes
- [ ] Repo is on GitHub with at least 3 commits
- [ ] You can `uv run python -c "import abtest; print(abtest.__version__)"`

---

## Week 1 — Design & diagnostics (10 hours)

### Step 1.1 — `design.py` (4 hours)

Four functions. The math you already know; write it cleanly.

```python
import numpy as np
from scipy import stats


def sample_size_proportions(
    baseline_rate: float,
    mde: float,
    alpha: float = 0.05,
    power: float = 0.80,
    n_variants: int = 1,
    relative: bool = True,
) -> int:
    """Sample size per group for a two-proportion test.

    Args:
        baseline_rate: Control conversion rate, e.g. 0.10.
        mde: Minimum detectable effect. Relative (0.05 = 5% lift) if
            relative=True, otherwise absolute (0.05 = 10pp -> 15pp).
        n_variants: Number of treatment arms. Applies a Bonferroni
            correction to alpha.

    Returns:
        Required sample size per group, rounded up.
    """
    ...
```

The formula:

```
n = (z_{α/2} + z_β)² · [p₁(1−p₁) + p₂(1−p₂)] / (p₂ − p₁)²
```

where `z_{α/2} = stats.norm.ppf(1 - alpha/2)` and `z_β = stats.norm.ppf(power)`.

The other three:

- `sample_size_means(std_dev, mde, alpha, power)` — `n = 2σ²(z_{α/2} + z_β)² / δ²`
- `minimum_detectable_effect(n, baseline_rate, alpha, power)` — invert the above; use `scipy.optimize.brentq` if algebra gets messy
- `experiment_duration(n_required, daily_users, allocation=0.5)` — returns days, and this is the function a PM will actually call

**The multiple-variants point matters.** With 3 treatment arms tested at α = 0.05 each, family-wise error is 1 − 0.95³ ≈ 14%. Apply Bonferroni (`alpha / n_variants`) and document it in the docstring.

### Step 1.2 — Tests that cross-validate (2 hours)

This is the habit that makes your tests worth something:

```python
import pytest
from statsmodels.stats.power import NormalIndPower
from statsmodels.stats.proportion import proportion_effectsize

from abtest.design import sample_size_proportions


def test_sample_size_matches_statsmodels():
    """Our implementation should agree with a trusted reference."""
    baseline, treatment = 0.10, 0.12
    effect = proportion_effectsize(treatment, baseline)
    expected = NormalIndPower().solve_power(
        effect_size=effect, alpha=0.05, power=0.80, ratio=1.0
    )
    ours = sample_size_proportions(baseline, mde=0.20, relative=True)
    assert ours == pytest.approx(expected, rel=0.02)


def test_smaller_effect_needs_more_users():
    assert sample_size_proportions(0.10, 0.05) > sample_size_proportions(0.10, 0.20)
```

### Step 1.3 — `diagnostics.py` (4 hours)

**Sample ratio mismatch** — the highest-value function in the whole package:

```python
def check_srm(
    observed_counts: dict[str, int],
    expected_ratios: dict[str, float] | None = None,
    threshold: float = 0.001,
) -> dict:
    """Chi-square test for sample ratio mismatch.

    A failing SRM means randomization or logging is broken. When this
    fails, the experiment should not be analyzed at all — the treatment
    effect estimate is contaminated by whatever caused the imbalance.

    Returns:
        dict with chi2, p_value, passed (bool), and observed vs expected
        proportions.
    """
    ...
```

Use `stats.chisquare(observed, expected)`. Note the threshold is 0.001, not 0.05 — SRM checks run constantly and you don't want false alarms, but a genuine SRM produces astronomically small p-values anyway.

Then:

- `run_aa_test(data, n_simulations=1000)` — repeatedly split identical data at random, run your analysis, confirm you get significance ≈ 5% of the time. Validates the whole pipeline.
- `winsorize(series, limits=(0.0, 0.01))` — with a docstring stating clearly that thresholds must be chosen *before* seeing outcomes.

### Week 1 definition of done

- [ ] 4 design functions + 3 diagnostic functions, all with docstrings
- [ ] At least 10 tests passing, including one cross-validation against statsmodels
- [ ] Pushed to GitHub

---

## Week 2 — Analysis (12 hours)

### Step 2.1 — `report.py` first (1 hour)

Define the output type before the functions that produce it:

```python
from dataclasses import dataclass


@dataclass
class ExperimentResult:
    metric_name: str
    control_value: float
    treatment_value: float
    absolute_effect: float
    relative_effect: float
    ci_lower: float
    ci_upper: float
    p_value: float
    method: str
    n_control: int
    n_treatment: int

    @property
    def is_significant(self, alpha: float = 0.05) -> bool:
        return self.p_value < alpha

    def is_practically_significant(self, threshold: float) -> bool:
        """Statistical significance is not a shipping decision."""
        return self.is_significant and abs(self.relative_effect) >= threshold

    def summary(self) -> str:
        """Human-readable one-paragraph summary."""
        ...
```

### Step 2.2 — Standard tests (3 hours)

- `welch_ttest(control, treatment)` — `stats.ttest_ind(..., equal_var=False)`. Default to Welch, always. Equal variance between arms is an assumption that's rarely true and almost never checked.
- `proportions_ztest(...)` — for binary metrics
- `mann_whitney(control, treatment)` — with a docstring that states plainly: this tests stochastic dominance, **not** a difference in means. Reporting it as "the means differ" is one of the most common errors in industry analysis.

### Step 2.3 — Bootstrap (3 hours)

```python
def bootstrap_ci(
    control: np.ndarray,
    treatment: np.ndarray,
    statistic: Callable = np.mean,
    n_resamples: int = 10_000,
    confidence: float = 0.95,
    random_state: int | None = None,
) -> tuple[float, float]:
    """Percentile bootstrap CI for the difference in a statistic.

    Use when the statistic has no clean analytical variance — medians,
    percentiles, trimmed means.
    """
    rng = np.random.default_rng(random_state)
    diffs = np.empty(n_resamples)
    for i in range(n_resamples):
        c = rng.choice(control, size=len(control), replace=True)
        t = rng.choice(treatment, size=len(treatment), replace=True)
        diffs[i] = statistic(t) - statistic(c)
    lo = (1 - confidence) / 2 * 100
    return np.percentile(diffs, lo), np.percentile(diffs, 100 - lo)
```

Vectorize it afterwards with `rng.choice(control, size=(n_resamples, len(control)))` — 10–50× faster, and a nice thing to mention in the README.

### Step 2.4 — The delta method (5 hours)

**This is the function that will impress people most.** Budget the full 5 hours; it's the hardest thing in the package.

**The problem:** your metric is clicks-per-session. You randomize by *user*, but you're analyzing *sessions*. Sessions from the same user are correlated, so treating them as independent gives variance estimates that are badly wrong — your nominal 95% CIs actually cover the true value around 80% of the time.

**The fix:** aggregate to the randomization unit (per user: numerator sum X, denominator sum Y), then apply the delta method to the ratio of means:

```
Var(X̄/Ȳ) ≈ (1/Ȳ²)·Var(X̄) − (2X̄/Ȳ³)·Cov(X̄,Ȳ) + (X̄²/Ȳ⁴)·Var(Ȳ)
```

where `Var(X̄) = Var(X)/n`, `Var(Ȳ) = Var(Y)/n`, `Cov(X̄,Ȳ) = Cov(X,Y)/n`.

```python
def ratio_metric_variance(numerator: np.ndarray, denominator: np.ndarray) -> float:
    """Delta-method variance for a ratio metric.

    Both arrays must be aggregated to the randomization unit — one row
    per user, not per session.
    """
    n = len(numerator)
    x_bar, y_bar = numerator.mean(), denominator.mean()
    var_x, var_y = numerator.var(ddof=1) / n, denominator.var(ddof=1) / n
    cov_xy = np.cov(numerator, denominator, ddof=1)[0, 1] / n
    return (
        var_x / y_bar**2
        - 2 * x_bar * cov_xy / y_bar**3
        + x_bar**2 * var_y / y_bar**4
    )
```

Then wrap it in `ratio_metric_test(control_num, control_den, treatment_num, treatment_den)` returning an `ExperimentResult`.

**How to verify it's right:** simulate data where you know the true ratio, run 1000 experiments, and check that your 95% CIs contain the truth ~95% of the time. If they do, the implementation is correct. This becomes simulation notebook #3 in Week 4.

### Week 2 definition of done

- [ ] `ExperimentResult` + 5 analysis functions
- [ ] Delta method verified by coverage simulation
- [ ] ~20 tests passing

---

## Week 3 — CUPED & the peeking simulation (12 hours)

### Step 3.1 — CUPED (4 hours)

Short function, big impact:

```python
def cuped_adjust(
    y: np.ndarray, x: np.ndarray
) -> tuple[np.ndarray, float, float]:
    """Adjust outcome using a pre-experiment covariate.

    Y_adj = Y - theta * (X - E[X]),  theta = Cov(X, Y) / Var(X)

    Variance is reduced by a factor of (1 - rho^2). At rho = 0.7 that
    is roughly half the variance — meaning half the users, or half the
    runtime, for equivalent power.

    Args:
        y: In-experiment metric.
        x: Pre-experiment covariate (usually the same metric measured
            before the experiment started).

    Returns:
        (adjusted_y, theta, variance_reduction_pct)
    """
    theta = np.cov(x, y, ddof=1)[0, 1] / np.var(x, ddof=1)
    y_adj = y - theta * (x - x.mean())
    reduction = 1 - np.var(y_adj, ddof=1) / np.var(y, ddof=1)
    return y_adj, theta, reduction * 100
```

**Critical constraint for the docstring:** the covariate must be measured *before* randomization. Using an in-experiment covariate biases the estimate, because treatment can affect the covariate itself.

Test: generate correlated `x` and `y` with known ρ, confirm the measured variance reduction ≈ ρ².

### Step 3.2 — The peeking simulation (8 hours)

**This produces the single most valuable image in your portfolio.** Notebook: `simulations/01_peeking.ipynb`.

The experiment: run many A/A tests — no real effect exists. Check for significance repeatedly and stop the moment you see p < 0.05. Measure how often you falsely "find" an effect.

```python
def simulate_peeking(
    n_simulations: int = 10_000,
    n_total: int = 10_000,
    n_looks: int = 10,
    baseline_rate: float = 0.10,
    alpha: float = 0.05,
    seed: int = 42,
) -> float:
    """Return the false positive rate when peeking n_looks times.

    Both arms are drawn from the same distribution, so every rejection
    is a false positive.
    """
    rng = np.random.default_rng(seed)
    checkpoints = np.linspace(n_total / n_looks, n_total, n_looks).astype(int)
    false_positives = 0

    for _ in range(n_simulations):
        control = rng.binomial(1, baseline_rate, n_total)
        treatment = rng.binomial(1, baseline_rate, n_total)
        for n in checkpoints:
            _, p = proportions_ztest(control[:n], treatment[:n])
            if p < alpha:
                false_positives += 1
                break  # the analyst stops here and ships

    return false_positives / n_simulations
```

Run for `n_looks` in `[1, 2, 5, 10, 20, 30]` and plot false positive rate against number of looks, with a dashed horizontal line at the nominal 0.05.

Expected shape: ~5% at one look, ~14% at five, ~20%+ at ten, approaching 30% at daily checks over a month.

**Performance note:** 10,000 simulations × 30 looks is 300,000 z-tests. Write your own z-test on numpy arrays rather than calling scipy in the inner loop, or drop to 2,000 simulations while developing and run the full 10,000 once at the end.

Save the figure to `simulations/figures/peeking.png` — it goes in the README.

Write 3–4 paragraphs under the chart explaining *why* this happens: each look is another opportunity to cross the threshold, and the sequence of test statistics is a random walk. Given enough looks, it will eventually wander past the critical value by chance alone.

### Week 3 definition of done

- [ ] CUPED implemented and tested
- [ ] Peeking simulation run at full scale, figure saved
- [ ] Notebook has narrative text, not just code cells

---

## Week 4 — Remaining simulations & README (10 hours)

### Step 4.1 — `02_cuped.ipynb` (3 hours)

Simulate pre/post pairs across ρ from 0.0 to 0.9. For each, plot:
- Achieved power with and without CUPED at fixed n
- Sample size required for 80% power, with and without

The takeaway line for your README: at a realistic ρ ≈ 0.6, CUPED cuts required sample size by roughly a third.

### Step 4.2 — `03_ratio_metrics.ipynb` (3 hours)

The coverage study from Week 2, written up properly. Simulate users with variable session counts, compute CIs both naively (per session) and via the delta method, and plot actual coverage against the nominal 95% as you increase the variance in sessions-per-user.

The naive line falling to ~80% while delta method holds at 95% is a compelling picture.

### Step 4.3 — The README (4 hours)

**Spend real time on this. Most people who look at the repo will read only this file.**

Structure:

```markdown
# abtest-toolkit

One sentence: what problem this solves.

## Why this exists

3–4 sentences. The problems: teams peek at results, misuse ratio metrics,
run underpowered tests. This package handles the design, diagnostics,
and analysis correctly.

## The peeking problem
[embed peeking.png]
Two paragraphs, with the headline number: checking daily for a month
turns a 5% false positive rate into roughly 30%.

## Quick start
[10 lines of code showing a realistic end-to-end usage]

## What's included
Short table: module | what it does

## Case study
Link to the memo, with the headline finding.

## Installation
## Development
## References
```

Lead with the problem and the finding. The tech stack goes near the bottom — a recruiter reads the first paragraph and the first image, and often nothing else.

### Week 4 definition of done

- [ ] Three simulation notebooks, each with narrative text
- [ ] README with embedded figures
- [ ] Repo looks presentable to a stranger

---

## Week 5 — Case study & decision memo (8 hours)

### Step 5.1 — The data (1 hour)

**Cookie Cats** (search Kaggle for "mobile games ab testing"). ~90,000 mobile game players, randomized to a progression gate at level 30 vs level 40, with day-1 and day-7 retention plus total game rounds played.

It's the best public option because the result is genuinely ambiguous — one metric points one way, another points differently — which forces a real judgment call rather than a rubber stamp.

### Step 5.2 — The analysis (4 hours)

Run the full sequence in `case_study/analysis.ipynb`, using **your own package** at every step:

1. `check_srm()` on the group counts — is the split what it should be?
2. Distribution inspection — plot rounds-played. It's heavily skewed, which justifies your later choices.
3. `proportions_ztest()` on day-1 and day-7 retention
4. `bootstrap_ci()` on rounds played, since the mean is a poor summary of a skewed distribution
5. `sample_size_proportions()` retrospectively — was this experiment adequately powered for the effect observed?
6. `is_practically_significant()` — is the effect large enough to justify a product change?

Using your own API here is the point. It shows the package works on real data and exposes any awkwardness in the interface, which you should then fix.

### Step 5.3 — The decision memo (3 hours)

`case_study/decision_memo.md`. **One page. No statistical jargon.**

```markdown
# Gate placement experiment — recommendation

## Recommendation
[Ship / don't ship / extend], in one sentence.

## What we found
2–3 sentences in plain language. Percentages, not p-values.

## How confident we are
2 sentences. Where the uncertainty is.

## Risk if we're wrong
2 sentences. What it costs.

## What I'd measure next
2–3 bullets.
```

**A hiring manager will read this more carefully than any of your code.** It's direct evidence you can turn statistics into a business decision, which is the actual job. Write it, leave it a day, then cut it by a third.

### Week 5 definition of done

- [ ] Case study notebook using your own package throughout
- [ ] One-page memo, jargon-free
- [ ] README links to both

---

## Week 6 — Sequential testing & publishing (10 hours)

This week is upside. Weeks 0–5 already give you a complete, defensible project — if you're short on time, skip to 6.2 and 6.3.

### Step 6.1 — `sequential.py` (5 hours)

Alpha-spending functions let you look at predetermined checkpoints while preserving the overall error rate. Pocock spends α evenly; O'Brien-Fleming is conservative early and permissive late.

```python
def obrien_fleming_boundaries(n_looks: int, alpha: float = 0.05) -> np.ndarray:
    """Z-score boundaries for each interim look.

    Spending function: alpha(t) = 2 * (1 - Phi(z_{alpha/2} / sqrt(t)))
    where t is the information fraction (n_so_far / n_planned).
    """
    ...
```

Then rerun the Week 3 peeking simulation *using these boundaries* and show the false positive rate returning to 5%. That's the payoff — problem demonstrated in Week 3, problem solved in Week 6. Add the before/after chart to the README.

Always-valid confidence sequences are a further stretch. Attempt only if everything else is done.

### Step 6.2 — CI (2 hours)

`.github/workflows/ci.yml`:

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3
      - run: uv sync --dev
      - run: uv run pytest --cov=abtest --cov-report=term-missing
      - run: uv run ruff check .
```

Add the passing badge to the README.

### Step 6.3 — Publish to PyPI (3 hours)

Register at pypi.org, create an API token, then:

```bash
uv build
uv publish
```

A candidate with `pip install abtest-toolkit` is in a different category from a candidate with a repo. It takes an afternoon and it's the cheapest credibility on this entire list.

---

## When you get stuck

**Package import errors.** Almost always the layout. Confirm `src/abtest/__init__.py` exists and run `uv sync` again.

**A statistical function gives wrong numbers.** Cross-check against `statsmodels` or `scipy` first — the reference implementation tells you whether the bug is in your formula or your inputs. If they agree and it still looks wrong, your test data is wrong.

**A simulation runs too slowly.** Drop to 500 iterations while developing. Vectorize the inner loop before scaling up. Never debug at full scale.

**You've lost momentum mid-week.** Ship the smaller version. A working percentile bootstrap beats an unfinished BCa bootstrap. Every module here has a simple version that's genuinely useful — build that first, improve later if time allows.

**A whole week slips.** Fine. Weeks 1, 2, 3, and 5 are the load-bearing ones. Week 4's notebooks can be thinner and Week 6 is optional. Don't abandon the project because the schedule slipped; the schedule is a guide, not a commitment.

---

## Resources

**The one book worth buying:** Kohavi, Tang & Xu, *Trustworthy Online Controlled Experiments* (2020). It is the reference for this entire domain, written by the people who built experimentation at Microsoft, Amazon, and Airbnb. Chapters 17–22 cover almost everything in this package.

**Free and directly useful:**
- Kohavi's published papers on experimentation pitfalls — searchable by name, several are freely available
- Evan Miller's writing on sample sizes and sequential testing at evanmiller.org
- The original CUPED paper by Deng, Xu, Kohavi & Walker (2013) — search the title "Improving the Sensitivity of Online Controlled Experiments by Utilizing Pre-Experiment Data"
- Engineering blogs at Netflix, Booking.com, and Spotify all have substantial experimentation write-ups

**Python packaging:** the official Python Packaging User Guide, plus the `uv` documentation at docs.astral.sh/uv.

---

## Summary

| Week | Hours | Output |
|---|---|---|
| 0 | 4–6 | Installable package, one passing test |
| 1 | 10 | Design + diagnostics, cross-validated tests |
| 2 | 12 | Analysis module including delta method |
| 3 | 12 | CUPED + peeking simulation |
| 4 | 10 | Two more simulations + README |
| 5 | 8 | Case study + decision memo |
| 6 | 10 | Sequential testing, CI, PyPI |

**Minimum viable version:** Weeks 0–3 plus the README. That's ~40 hours and already a strong portfolio piece.

**The one thing not to cut:** the peeking simulation. It's the most memorable artifact in the project and the one most likely to come up in an interview.
