# abtest-toolkit — 30-Day Plan

Compressed from the 6-week [build guide](abtest-toolkit-build-guide.md) into 30 days, ~2h/day average (~60h total). Start: Sat Aug 22, 2026 → Sun Sep 20, 2026.

## What's cut vs. the full guide

- **Always-valid confidence sequences** — explicitly a stretch item in the guide ("attempt only if everything else is done"). Skipped entirely.
- **Deep vectorization/perf polish** — bootstrap gets vectorized once (Day 12), simulations run at a size that finishes in reasonable time, no further optimization pass.
- **Interface refactoring / extra docstring polish** — write it once, correctly, don't circle back.
- **Sequential testing (Day 26)** is the one item still in scope that's "upside, not required" per the guide. If you're behind schedule by Day 20, cut this day and go straight to CI/PyPI — the package is already complete and defensible without it.

Everything else (design, diagnostics, analysis, delta method, CUPED, the peeking simulation, README, case study, CI, PyPI) stays, because those are the load-bearing pieces per the guide's own Week 1/2/3/5 emphasis.

## Weekly shape

| Week | Days | Focus | Hours |
|---|---|---|---|
| A | 1–7 | Setup + `design.py` + `diagnostics.py` | ~14h |
| B | 8–14 | `report.py` + `analysis.py` + delta method | ~14h |
| C | 15–21 | CUPED + peeking sim + other sims + README | ~15h |
| D | 22–28 | Case study + memo + sequential + CI + PyPI | ~13h |
| Buffer | 29–30 | Polish, verify, ship | ~3h |

## Daily plan

| Day | Date | Task | Est. |
|---|---|---|---|
| 1 | Sat Aug 22 | Finish Week 0 setup: `uv add` deps, fix `pyproject.toml` description, create `tests/`, `simulations/`, `case_study/`, six module stub files | 2h |
| 2 | Sun Aug 23 | `hello()` + `test_design.py`, `uv run pytest` → 1 passed, git add/commit, create GitHub repo, rename branch to `main`, push | 1.5h |
| 3 | Mon Aug 24 | `design.py`: `sample_size_proportions`, `sample_size_means` | 2h |
| 4 | Tue Aug 25 | `design.py`: `minimum_detectable_effect`, `experiment_duration`, Bonferroni correction for `n_variants` | 2h |
| 5 | Wed Aug 26 | `tests/test_design.py`: cross-validate `sample_size_proportions` against `statsmodels` | 2h |
| 6 | Thu Aug 27 | `diagnostics.py`: `check_srm` (chi-square) + tests | 2h |
| 7 | Fri Aug 28 | `diagnostics.py`: `run_aa_test`, `winsorize` + tests. Commit "Week 1 done" | 2h |
| 8 | Sat Aug 29 | `report.py`: `ExperimentResult` dataclass (`is_significant`, `is_practically_significant`, `summary`) | 1.5h |
| 9 | Sun Aug 30 | `analysis.py`: `welch_ttest`, `proportions_ztest` | 2h |
| 10 | Mon Aug 31 | `analysis.py`: `mann_whitney` + tests, docstring on stochastic-dominance caveat | 2h |
| 11 | Tue Sep 1 | `analysis.py`: `bootstrap_ci` (loop version) + tests | 2h |
| 12 | Wed Sep 2 | Vectorize `bootstrap_ci`. Commit | 1.5h |
| 13 | Thu Sep 3 | `variance.py`: `ratio_metric_variance` (delta method) | 2.5h |
| 14 | Fri Sep 4 | `ratio_metric_test` wrapper returning `ExperimentResult` + basic coverage check | 2h |
| 15 | Sat Sep 5 | `variance.py`: `cuped_adjust` + test with known ρ | 2h |
| 16 | Sun Sep 6 | `simulations/01_peeking.ipynb`: `simulate_peeking` + own numpy z-test | 2.5h |
| 17 | Mon Sep 7 | Run peeking sim across `n_looks`, plot false-positive rate, save `figures/peeking.png` | 2h |
| 18 | Tue Sep 8 | Write narrative paragraphs in peeking notebook. Commit — this is the flagship artifact | 1.5h |
| 19 | Wed Sep 9 | `simulations/02_cuped.ipynb`: power/sample-size vs ρ sweep (thin version) | 2h |
| 20 | Thu Sep 10 | `simulations/03_ratio_metrics.ipynb`: naive vs. delta-method coverage study | 2h |
| 21 | Fri Sep 11 | `README.md`: full structure, embed peeking figure, quick start, module table | 2.5h |
| 22 | Sat Sep 12 | Cookie Cats data: `check_srm` on group counts, inspect rounds-played distribution | 2h |
| 23 | Sun Sep 13 | `proportions_ztest` on day-1/day-7 retention, `bootstrap_ci` on rounds played | 2h |
| 24 | Mon Sep 14 | Retrospective `sample_size_proportions`, `is_practically_significant` judgment call | 1.5h |
| 25 | Tue Sep 15 | Draft `decision_memo.md` (one page, no jargon) | 1.5h |
| 26 | Wed Sep 16 | `sequential.py`: `obrien_fleming_boundaries`, rerun peeking sim with boundaries, before/after chart — **first to cut if behind** | 2.5h |
| 27 | Thu Sep 17 | `.github/workflows/ci.yml`, fix any lint/test failures, add badge | 2h |
| 28 | Fri Sep 18 | Register PyPI, `uv build && uv publish` | 1.5h |
| 29 | Sat Sep 19 | Polish: commit history sanity check, final README pass, link case study + memo | 2h |
| 30 | Sun Sep 20 | Final verify: full test suite green, `pip install abtest-toolkit` works from a clean env | 1h |

## Definition of done (whole project)

- [ ] `uv run pytest` passes with cross-validated tests for design/analysis functions
- [ ] Peeking simulation figure in README with narrative
- [ ] Case study notebook uses the package end-to-end + one-page decision memo
- [ ] CI green on GitHub, package published on PyPI
- [ ] Commit history readable — one commit per function/change, no "update" messages
