# Plan: Does an Inverted Yield Curve Reliably Predict Recessions?

## Question
Does a negative 10-year minus 2-year Treasury spread (a "yield curve inversion")
reliably precede U.S. recessions, or is it a noisier signal than its reputation
suggests — with variable lead times and at least some false alarms?

**My prior expectation:** noisy. I expect inversions to precede *most* recessions,
but with an inconsistent lag (sometimes 6 months, sometimes close to 2 years) and
possibly at least one case that doesn't cleanly fit.

## Data sources
All pulled live from the FRED API using the `fredapi` package (`Fred.get_series`),
so the notebook re-pulls fresh data every run — no CSVs saved to the repo.

| Series  | What it is                                     | Frequency |
|---------|-------------------------------------------------|-----------|
| `GS10`  | 10-Year Treasury Constant Maturity Rate          | Monthly   |
| `GS2`   | 2-Year Treasury Constant Maturity Rate           | Monthly   |
| `USREC` | NBER-based Recession Indicator (1 = recession)   | Monthly   |

Sample period: **1990-01 to present**. This keeps the window recent enough to
feel current while still covering four NBER recessions (1990–91, 2001, 2007–09,
2020) plus the 2022–23 inversion episode, whatever it turns out to have preceded
by the time we pull the data — that last one is a live, well-known test case for
the "noisy" story either way.

API key handling: `FRED_API_KEY` lives in a `.env` file (gitignored), loaded with
`python-dotenv`'s `load_dotenv()`, and passed to `Fred(api_key=os.getenv("FRED_API_KEY"))`
— never typed into the notebook itself.

## Cleaning steps
1. Pull all three series as `pd.Series` objects (each comes back with a
   `DatetimeIndex`, one value per month).
2. Trim all three to 1990-01 onward.
3. Compute `spread = GS10 - GS2` — plain Series subtraction, aligned on the
   shared date index.
4. Build `inverted = spread < 0` (a boolean Series) and `spread_lag12 =
   spread.shift(12)` (spread from 12 months earlier, lined up with today's
   recession status) — the same `.shift()` used for month-over-month growth in
   the matplotlib lecture, just with a 12-month lag instead of 1.
5. Sanity checks before analysis: no duplicate dates, `USREC` only takes values
   {0,1}, no unexpected gaps in the monthly index.

## Data validation checks (unit tests)
Run immediately after pulling each series from FRED, before any cleaning or
analysis, so a bad API response fails loudly instead of quietly corrupting
downstream results:

- `assert gs10.index.is_unique` and the same for `gs2`, `usrec` — no duplicate dates.
- `assert usrec.dropna().isin([0, 1]).all()` — recession indicator is only 0 or 1.
- `assert gs10.index.is_monotonic_increasing` — dates are in order (no scrambled pull).
- After merging: `assert len(spread) == len(usrec_aligned)` — the two series line
  up one-to-one, no silent row loss from a bad merge.
- Spot check: `assert spread.loc["2006-11"].item() < 0` — the 10y–2y spread is
  known to be negative in November 2006, ahead of the Great Recession; if this
  fails, something upstream (units, sign, wrong series ID) is wrong.
- Second spot check: `assert spread.loc["2023-07"].item() < 0` — same test on the
  2022–24 episode, whose deepest month is unambiguous (−0.93).
- `assert spread.between(-5, 5).all()` — the spread should stay within a
  plausible range in percentage points; anything wildly outside that suggests
  bad units or a data error.

> **Revised 2026-09-28 — the original spot check was wrong.** This section first
> asserted `spread.loc["2007-09"] < 0`, on the belief that the curve was inverted
> in September 2007. It was not: the spread that month is **+0.51**. The inversion
> ahead of the 2007–09 recession ran from 2006-02 to 2007-05, and by September
> 2007 the Fed had begun cutting and the curve had re-steepened. Caught while
> building the validation cell, when the assert failed against live FRED data —
> which is exactly what a spot check is for, even though here it was the check
> that was wrong rather than the data. Moved to 2006-11 (−0.14), the deepest
> month of that inversion, and a second check on 2023-07 was added.

## Charts
1. **Line chart** of the spread over time, a horizontal line at 0, and recession
   bars shaded using `USREC` + `fill_between` — direct reuse of the week-4
   recession-shading skill, applied to a new series.
2. **Annotated chart** marking each inversion's start date and the following
   recession's start date with `ax.annotate` arrows, so the lead time is visible
   at a glance for each episode.
3. **Bar chart** of "months between inversion onset and recession onset" for each
   episode — this is the chart that will visually show whether the lag is
   consistent (supports "reliable") or scattered (supports "noisy").

## Tests
1. **Two-sample t-test** (`scipy.stats.ttest_ind`): compare `spread_lag12` in
   months that were recession months vs. months that weren't. If the
   inverted-12-months-ago group has a meaningfully lower mean, that's evidence
   for the signal.
2. **Logistic regression** (`statsmodels.api.Logit`): `USREC ~ spread_lag12`,
   asking whether the spread 12 months ago predicts recession now. One lag
   only, kept simple. This goes a little beyond what's been lectured so far —
   `statsmodels` has only been mentioned in the toolkit overview, not taught in
   a lecture cell — so every argument and output field gets explained when we
   get there, and this is flagged as self-directed in the "how I used Claude
   Code" writeup.

**Limitation to flag up front:** with only four full recessions in the 1990–
present window, the regression is best read as a compact description of the
same pattern the charts show, not a rigorously powered causal test. This will
be stated plainly in the conclusion rather than overstating the p-value.

## What would support vs. contradict my expectation
- **Supports "noisy":** lead times vary meaningfully across episodes (e.g. some
  6 months, some 18+), and/or the regression coefficient is not statistically
  significant, and/or the 2022–23 inversion behaves differently from the
  earlier ones.
- **Contradicts "noisy" (i.e., signal is reliable):** all episodes precede
  recession within a similar, narrow window, and the regression shows a
  strong, significant relationship.
