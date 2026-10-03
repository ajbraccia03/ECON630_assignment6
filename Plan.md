# Plan: Does an Inverted Yield Curve Reliably Predict Recessions?

## Question

Does the historical indicator of a "yeild curve inversion" (negative 10yr minus 2yr treasury spread) reliably precede U.S. recessions, or is it a noiser signal that its reputation suggests? 

**expectation going in** my gut reaction is that the signal is a historical signal because there is evidence. However, as the american economy has changed over the last century I expect the signal to be little noisy. as a result, i expect inversions to precede **most** recessions but with an inconsistent lag time. 

## Data sources
All data is pulled live from FRED API using the `fredapi` package so that the notebook can pull fresh data each run not use CSVs. the series below are what are being pulled from fred: 

| Series  | What it is                                     | Frequency |
|---------|-------------------------------------------------|-----------|
| `GS10`  | 10-Year Treasury Constant Maturity Rate          | Monthly   |
| `GS2`   | 2-Year Treasury Constant Maturity Rate           | Monthly   |
| `USREC` | NBER-based Recession Indicator (1 = recession)   | Monthly   |

Sample period: **1976-06 to present**. This sample period is used because the data does not exist prior to 1976-06 in fred. moreover, the indicator tests the spread across several monetary policy regimes like the late 70s period of inflation, disinflation, and the post 1990-era. as a result of this sample period it means in the data we have 6 NBER recessions. 


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
Runs of `spread < 0` seperated by a gap of three months or less are one episdode (`MAX_GAP = 3`). furthermore, single month dips are kept rather than filtered out. 

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

24 months is used and sits just above the longest lead in the sample, so the window is a choice,not a finding. 

## Charts
1. **Line Chart** of the spread over time, a horizontal line at 0, and the recession 
   bars shaded using `USREC` + `fill_between` which is a direct reuse of the week-4 recession shading skill we applied to a different series. 
2. **Bar Chart** of months between inversion and recession onset. ther is one row 
   per **recession**. this is the chart that clearly shows whether the lag is consistent or scattered. false alarms appear below the recessions in red with a dashed line at the 24th month line. a recession with no inversion has no bar and a text row. 


## Test
**Logistic regression** (`statsmodels.api.Logit`): `USREC ~ spread_lag12`, asks whether the spread 12 months ago from present day predicts recession now. use one lag only, keeps it simple, fit once with HAC (Newey and West), standard erros asking whether the spread 12 months ago predicts recession now. One lag only, (`cov_type="HAC", cov_kwds={"maxlags": 12}`), because recession months come in runs. explain each argument and ouput in notebook since this goes a little beyond the current level of the class. 

## Limitations to flag 
with six recession samples the regression is best read as a compact description of the same patterns in the charts 

## What would support vs. contradict my expectation

- **Supports "Noisy"** is the lead times are meaningful across episodes (some 6 months, others 18 or more), and/or the regression coefficient is not statistically significant. moreover, at least one recession has no inversion in the prior 24 months; and/or one inversion is a false alarm.  

- **signal is reliable and "Noisy" is contradicted** all the episodes precede recession within a similar, narrow window, and the regression shows a significant relationship. 

