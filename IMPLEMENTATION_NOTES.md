# HW3 implementation and verification

## Scope and sources

The local `README.md` is authoritative. Parts B, C, and D are implemented;
the student is completing Part A independently. Instructor tests are unchanged.
Use Python 3.12 and the tracked `uv.lock` through `uv run`.

## Calculations

- `src/rnd.py`: recover the risk-neutral CDF as `1 + exp(r*tau) * dc/dk`,
  then retain the supplied isotonic regression and clipping.
- `src/svix.py`: select puts below the normalized forward and calls above it,
  integrate the resulting continuous OTM price curve with the trapezoidal
  rule, and multiply by `2 / (tau * R_f)`. Integrating over the entire grid
  includes the interval that straddles the forward.
- `src/crash_prob.py`: the log-investor probability is
  `alpha * Q(alpha) / R_f - put(alpha)`.
- `src/pull_optionmetrics.py`: select the supplied security IDs, maturities,
  and inclusive date window; preserve all deltas and both puts and calls.
- `dodo.py`: `latest` runs settings, both pulls, the monitor, and the page
  builder in order, always runs when requested, and does not duplicate
  other tasks' output targets.
- `chartbook.toml`: document the executed monitor notebook and history CSV.
- `.github/workflows/deploy_pages.yml`: daily 14:30 UTC schedule, repository
  secret, write permission, `uv run doit latest`, and `gh-pages` destination.

SVIX is interpolated in total variance to exactly 30 days. Crash probabilities
use the nearest usable expiry (`days_used`), and refer to the terminal return,
not the probability of crossing a barrier before expiry. For futures options,
moneyness uses the discounted forward as the spot. The OptionMetrics tail uses
the instructor's flat-volatility extrapolation beyond observed strikes.

## Local verification (November 8, 2026)

- `uv run pytest -q --ignore=src/test_github_skills.py
  --ignore=src/test_published_site.py`: **41 passed**, including all three
  historical/data-backed tests, with no skips.
- Ruff passes on the five edited Python files. `git diff --check` passes.
- The explicit local build succeeded:
  `uv run doit config pull compute_history monitor_chart monitor_page
  single_names run_notebooks build_chartbook_site`.
- All four notebooks executed every code cell without errors. The ChartBook
  site and monitor page were generated; the monitor was inspected in a browser.
- A subsequent explicit build reported every task up to date. Notebook 04
  needed one intermediate rerun because its optional historical dependencies
  first became available during the initial build.
- Databento estimates were all zero. The initial pull contains 14,648 option
  settlement rows over November 2–6, 2026. FRED initially ended November 5;
  the November 6 VIX is missing rather than filled with an invented value.
- WRDS yielded 1,522,316 surface rows, January 4, 1996–August 29, 2025.
  The 2026 annual tables are not available; the supplied pull reports this.
- Produced 29,742 index-probability rows and 2,818 single-name bound rows.
  Single-name bounds are finite, in [0, 1], ordered around the risk-neutral
  probability, and contain no duplicate observation keys.
- Daily SVIX comparison: 4,038 matched dates; correlation 0.9975; median
  absolute difference 0.29 volatility points. The separate graded
  month-end comparison also passes.
- Monthly crash comparison: 193 months; correlation 0.95629; median absolute
  difference 0.0026777 (0.26777 percentage points).

## Deployment

Final full-suite verification on November 9: **42 passed, 2 failed, no skips**.
The only failures are the two student-owned GitHub Skills completion flags.
Report: `_output/final_test_results.xml`. All 17 graded points in Parts B–D
pass locally. The course runner independently confirmed **17/19** on commit
`6b1146b` (run `42642766898`): B=5/5, C=8/8, D=4/4; only Part A fails.
An additional full default run after the date rolled over was stopped during
repeated Databento HTTP 500/stream retries. This was a redundant refresh;
the earlier complete local build and the cloud refresh both succeeded.

Public source repository: https://github.com/Weijie-Shen/svix-crash-monitor

The `public` remote points there. The private `origin` remains the grading
repository. The Databento key is stored as the public repository's encrypted
Actions secret, not in tracked files. Raw data, executed notebooks, local
ChartBook output, and credentials are excluded from Git.

Live monitor: https://weijie-shen.github.io/svix-crash-monitor/

The first deployment (Actions run `42567490299`) and GitHub Pages build
succeeded. The public page was verified in a browser before setting
`src/published_site.py` to completed. The four-week backfill from October 12
succeeded (run `42567801431`): 20 unique sessions through November 6, no
duplicate dates, and the expected ordering of the three crash probabilities.
The subsequent deployment `42642761551` successfully restored this history,
ran the daily update, and published the 20-session page to `gh-pages`.

The local `uv run doit latest` also succeeded. It found no new sessions and
left the history byte-for-byte unchanged (five rows, five unique dates).

## Operational notes

- Plain `uv run doit` includes all tests, so it cannot be fully green before
  deployment and Part A are genuinely completed. Do not alter tests or mark
  completion flags merely to obtain a green run.
- Follow the local README for date overrides and the backfill workflow's
  `start_date` input. Some online guidance differs from this local version.
- `monitor.py` computes only sessions missing from history. If a formula is
  changed after building history, deliberately regenerate the affected rows;
  an ordinary backfill will not replace already-known dates.
- The daily pull covers four calendar days to bridge weekends and short
  outages. A longer gap needs the backfill workflow.
- Backfill and deployment share a concurrency group. Avoid starting backfill
  near 14:30 UTC or pushing source while it runs, because deployment can
  cancel the in-progress backfill. Redeploy after backfill to refresh the page.
