# Plan: Does an Inverted Yield Curve Reliably Predict Recessions?

Revised 2026-10-01: revision history removed for submission. The method below is the final one. Earlier versions are in git.

## Question

Does the historical indicator of a "yeild curve inversion" (negative 10yr minus 2yr treasury spread) reliably precede U.S. recessions, or is it a noiser signal that its reputation suggests? 

**expectation going in** my gut reaction is that the signal is a historical signal because there is evidence. However, as the american economy has changed over the last century I expect the signal to be little noisy. as a result, i expect inversions to precede **most** recessions but with an inconsistent lag time. 

## Data sources
All data is pulled live from the FRED API using the `fredapi` package so that the notebook pulls fresh data everyrun -- no CSVs. the following series are what the notebook pulls from FRED.

| Series  | What it is                                     | Frequency |
|---------|-------------------------------------------------|-----------|
| `GS10`  | 10-Year Treasury Constant Maturity Rate          | Monthly   |
| `GS2`   | 2-Year Treasury Constant Maturity Rate           | Monthly   |
| `USREC` | NBER-based Recession Indicator (1 = recession)   | Monthly   |

Sample period: **1976-06 to present**. This is the earliest possible start date,
because FRED's `GS2` series does not exist before 1976-06. It also tests the
indicator across several monetary policy regimes: the late-1970s inflation, the
Volcker disinflation, and the post-1990 era. That means six NBER recessions
(1980, 1981–82, 1990–91, 2001, 2007–09, 2020) plus the 2022–24 inversion episode.

API key handling: `FRED_API_KEY` lives in a `.env` file (gitignored), loaded with
`python-dotenv`'s `load_dotenv()`, and passed to `Fred(api_key=os.getenv("FRED_API_KEY"))`
— never typed into the notebook itself.

## Cleaning steps
1. Pull all three series as `pd.Series` objects (each comes back with a
   `DatetimeIndex`, one value per month).
2. Trim all three to 1976-06 onward.
3. Compute `spread = GS10 - GS2` — plain Series subtraction, aligned on the
   shared date index. An inversion is `spread < 0`.
4. Build `inverted = spread < 0` (a boolean Series) and `spread_lag12 =
   spread.shift(12)` (spread from 12 months earlier, lined up with today's
   recession status) — the same `.shift()` used for month-over-month growth in
   the matplotlib lecture, just with a 12-month lag instead of 1.
5. Spot check before analysis: `assert spread.loc["2006-11"].item() < 0`. The
   10y–2y spread is known to be negative in November 2006, ahead of the Great
   Recession. If this fails, something upstream (sign, units, wrong series ID)
   is wrong.

## Inversion episodes
Runs of `spread < 0` separated by a gap of **three months or less** are one
episode (`MAX_GAP = 3`). Single-month dips are kept rather than filtered out.

## Lead times and false alarms
One constant, `SIGNAL_WINDOW = 24`, is used in both directions:

- **Lead time:** each recession gets one lead time, measured from the first month
  of the most recent episode that started in the **24 months before** the
  recession to the recession's first month. This holds whether that episode has
  already ended or is still negative when the recession starts.
- **No inversion:** a recession with no episode starting in the 24 months before
  it gets no lead time and is recorded as "no inversion".
- **False alarm:** an episode with no recession beginning within **24 months of
  its onset** is a false alarm. False alarms are never matched to a recession.
- An episode that started before the same recession as a later episode is
  **superseded**; only the most recent one sets the lead.

24 sits just above the longest lead in the sample, so the window is a choice,
not a finding, and the conclusion says so.

## Charts
1. **Line chart** of the spread over time, a horizontal line at 0, and recession
   bars shaded using `USREC` + `fill_between` — direct reuse of the week-4
   recession-shading skill, applied to a new series.
2. **Per-episode timeline** on a calendar axis: one row per episode, a bar over
   its inverted months, and an arrow from each signal episode's onset to the
   recession it was matched to. Superseded episodes are faded; false alarms get
   no arrow and are labeled with how long they went without a recession.
3. **Bar chart** of "months between inversion onset and recession onset", one row
   per **recession**. This is the chart that will visually show whether the lag
   is consistent (supports "reliable") or scattered (supports "noisy"). False
   alarms appear below the recessions as hatched rows, with a dashed line at the
   24-month window. A recession with no inversion gets a text row and no bar.

## Test
**Logistic regression** (`statsmodels.api.Logit`): `USREC ~ spread_lag12`,
asking whether the spread 12 months ago predicts recession now. One lag only,
kept simple, fit once with HAC (Newey-West) standard errors
(`cov_type="HAC", cov_kwds={"maxlags": 12}`), because recession months come in
contiguous runs and are not independent observations. This goes a little beyond
what's been lectured so far — `statsmodels` has only been mentioned in the
toolkit overview — so every argument and output field gets explained in the
notebook.

**Limitation to flag up front:** with six recessions in the sample, the
regression is best read as a compact description of the same pattern the charts
show, not a rigorously powered causal test. This will be stated plainly in the
conclusion rather than overstating the p-value.

## What would support vs. contradict my expectation
- **Supports "noisy":** lead times vary meaningfully across episodes (e.g. some
  6 months, some 18+), and/or the regression coefficient is not statistically
  significant, and/or the 2022–23 inversion behaves differently from the
  earlier ones.
- **Contradicts "noisy" (i.e., signal is reliable):** all episodes precede
  recession within a similar, narrow window, and the regression shows a
  strong, significant relationship.
