# Plan: Does an Inverted Yield Curve Reliably Predict Recessions?

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
because FRED's `GS2` series does not exist before 1976-06. It is also important to
test the indicator across multiple monetary policy regimes to show how reliable it
is in general. The window now covers the late-1970s inflation, the Volcker
disinflation, and the post-1990 era. That means six NBER recessions (1980,
1981–82, 1990–91, 2001, 2007–09, 2020) plus the 2022–24 inversion episode.

> **Revised 2026-09-29 — sample moved from 1990-01 to 1976-06.** The plan
> originally used **1990-01 to present**, "recent enough to feel current while
> still covering four NBER recessions." The analysis was completed and pushed on
> that sample. It was changed because that window covers only one monetary
> policy regime, and 1976-06 is the earliest date the data allows. There was a
> second, unplanned problem: the 1990-01 start cut the 1989 inversion in half.
> The spread was negative from 1989-01 to 1989-09, before the old sample began,
> so the old analysis saw only its 1990-03 tail. Every stage of the notebook is
> being redone on the new sample. Results quoted elsewhere in this plan from the
> 1990 sample are marked as such.

API key handling: `FRED_API_KEY` lives in a `.env` file (gitignored), loaded with
`python-dotenv`'s `load_dotenv()`, and passed to `Fred(api_key=os.getenv("FRED_API_KEY"))`
— never typed into the notebook itself.

## Cleaning steps
1. Pull all three series as `pd.Series` objects (each comes back with a
   `DatetimeIndex`, one value per month).
2. Trim all three to 1976-06 onward. *(Revised 2026-09-29: was 1990-01; see
   Sample period.)*
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
- Third spot check: `assert spread.loc["1981-05"].item() < 0`. That is the
  deepest month (−1.36) of the inversion ahead of the 1981–82 recession, and it
  tests the Volcker era. *(Added 2026-09-29 with the move to the 1976-06 sample.
  The first two checks both fall after 2006, so nothing tested the new, earlier
  part of the sample.)*
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

## Defining an inversion episode
Added 2026-09-28, while building the charts — the original plan measured lead
times per "episode" without saying what separates one episode from the next, and
the spread crosses zero more often than there are distinct inversions.

Runs of `spread < 0` separated by a gap of **three months or less** are merged
into a single episode (`MAX_GAP = 3`), and single-month dips are kept rather
than filtered out. Keeping them is the consequential half of the rule: 1998-06
is a one-month dip, and discarding it would throw away the clearest false alarm
in the sample. Lead time is measured from the **first** month of an episode to
the first month of the next recession.

> **Revised 2026-09-29.** This paragraph originally said "1990-03 and 1998-06 are
> both one-month dips, and discarding them would throw away one real signal and
> the clearest false alarm." With the sample starting in 1976-06, 1990-03 is no
> longer a lone signal. It is a one-month dip five months after the 1989-01 to
> 1989-09 inversion ends. The gap is longer than `MAX_GAP`, so the rule makes it
> a separate episode before the same 1990 recession. The 1976 sample raises two
> questions the 1990 sample never did. Both are settled under "Assigning a lead
> time to each recession" below:
> (a) when two episodes precede the same recession, which one's lead time
> counts; and (b) how to handle the 1978–82 inversions, which continue *into*
> the 1980 and 1981–82 recessions rather than ending before them.

`MAX_GAP` is a named constant so the sensitivity of the lead times to this
choice can be checked.

### Assigning a lead time to each recession
Added 2026-09-29, at the start of the charts-stage redo. This settles questions
(a) and (b) above. The rule, in my words:

> For each recession, find the most recent negative-spread stretch that started
> before the recession began — whether that stretch has already ended or is still
> negative by the time the recession starts. The lead time is the gap between
> that stretch's onset and the recession's start. If the identified stretch is
> the same one already used for an earlier recession (no new inversion happened
> in between), don't produce a second bar — note it as a continuation instead.

A "stretch" is an episode as defined above (`MAX_GAP = 3`). Lead times are now
counted **per recession, not per episode**, so the lead-time chart has at most
one bar per recession.

What the rule does on the 1976-06 sample:
- **1980:** from 1978-09, **17 months**. The stretch continues into the recession.
- **1981–82:** from 1980-09, **11 months**. The stretch continues into the recession.
- **1990–91:** from 1990-03, **5 months**. This one-month dip (−0.04) is more
  recent than the 1989-01 to 1989-09 inversion, so it sets the lead.
- **2001:** from 2000-02, **14 months**. It is more recent than the 1998-06 dip.
- **2007–09:** from 2006-02, **23 months**.
- **2020:** **no inversion**, no bar. No stretch started in the 24 months before
  2020-03. The most recent one, 2006-02, began 169 months earlier.
  *(Revised 2026-09-29: this originally said "continuation, no bar. The most
  recent stretch is 2006-02, which is already the 2007–09 bar." See "Symmetric
  window" below.)*
- **Not counted as any recession's signal:** 1989-01, superseded by the later
  1990-03 stretch before the same recession. 1998-06 and 2022-07 are **false
  alarms** (see below).
  *(Revised 2026-09-29: this originally said 1998-06 was superseded and the
  2022-07 stretch was "still open because no recession has followed it yet.")*

### False alarms
Added 2026-09-29, during the charts-stage redo. The rule, in my words:

> An inversion episode is a **false alarm** if no recession begins within
> **24 months of its onset**. False alarms are left out when assigning lead
> times, so a later recession can never be matched to one.

The window is a named constant, `SIGNAL_WINDOW = 24`, like `MAX_GAP`. An episode
with less than 24 months of data since its onset is **open**, meaning it can't be
judged yet. No episode is open in the current data.

What the rule does on the 1976-06 sample:
- **2022-07:** false alarm. It has gone 49 months with no recession. The spread
  turned positive in 2024-09 and has stayed positive for 24 months.
- **1998-06:** false alarm. The next recession came 34 months later. This used to
  be labeled "superseded" by the 2000-02 stretch.
- **No lead time changes.** Every real lead is 23 months or less, so the rule
  removes no bars.

Why the rule was needed: without it, the 2022 stretch stayed open forever.
Under the lead-time rule above, a recession starting at any point in the future
would have been matched to it as a lead of 50+ months.

**Caveat, stated plainly:** 24 sits just above the longest real lead in the
sample (23 months, 2006-02 to 2008-01). The window could be accused of being
fitted to the data. That is why it is a named constant, and the conclusion
should say so.

> **Correction, 2026-09-29.** I first asked for this rule on the belief that the
> plan already said the signal ends around 24 months. It didn't. No 24-month
> limit had ever been written here. The only related text was the example lags
> in "What would support vs. contradict my expectation" ("some 6 months, some
> 18+"). A 24-month window had been proposed in conversation and not taken up.
> The rule above is new, not a restatement.

### Symmetric window
Added 2026-09-29, during the charts-stage redo. The same `SIGNAL_WINDOW` also
limits the recession side. When a lead time is assigned, a recession is matched
only to an episode that started in the **24 months before it**. A recession with
no such episode is recorded as **no inversion**. One constant is used in both
directions: an episode has 24 months to be followed by a recession, and a
recession looks back 24 months for an episode.

> **Revised 2026-09-29 — why the backward search was capped.** In the rule
> above, the recession-side search looked backward with no distance limit.
> That is how 2020 got matched to the 169-month-old 2006-02 episode as a
> "continuation". The sensitivity check below exposed the same flaw more
> sharply. At `SIGNAL_WINDOW = 18`, 2006-02 became a false alarm, and the 2008
> recession was then matched to the 2000-02 episode, 95 months earlier. The
> forward rule had a limit and the backward one didn't.
>
> What changed: 2020 went from "continuation" to "no inversion". Nothing else
> moved, and every bar is the same.
>
> A side effect: with the same window both ways, a continuation now requires a
> single episode to start within 24 months before two different recessions.
> That never happens in this sample, so the continuation clause of the lead-time
> rule stays in the rule but never applies.

### Sensitivity of the window
Checked 2026-09-29. Both false alarms were re-classified under a tighter and a
looser window. Each run re-applies the full rule in both directions:

| Episode | Months to next recession | 18 | **24** | 36 |
|---|---|---|---|---|
| 1998-06 | 34 | false alarm | false alarm | superseded |
| 2022-07 | 49, no recession | false alarm | false alarm | false alarm |

- **2022-07 is a false alarm under all three.** That result is robust.
- **1998-06 depends on the window.** It is a false alarm at 24 but only
  superseded at 36. The conclusion should present it as a choice, not a finding.

The notebook computes this table from the data rather than typing it in, and an
assert checks that its 24-month column matches the main classification.

This answers (b): stretches that run into a recession stay whole, because the
lead is measured from the onset. The continuation clause never triggers between
1980 and 1981–82 at `MAX_GAP = 3`. Their stretches are separated by a 4-month
gap (1980-04 to 1980-09). At `MAX_GAP = 4` or more they would merge, and the
1981–82 recession would become a continuation.

> **Revised 2026-09-29.** This replaces the rule that measured a lead time for
> every episode from its onset to the next recession. On the 1976 sample, that
> rule counted the 1990–91 and 2001 recessions twice each (1989-01 *and*
> 1990-03; 1998-06 *and* 2000-02). The earlier notebook had the same flaw: its
> text called 1998 a false alarm, but its lead-time chart still counted it as a
> 34-month lead.

## Charts
1. **Line chart** of the spread over time, a horizontal line at 0, and recession
   bars shaded using `USREC` + `fill_between` — direct reuse of the week-4
   recession-shading skill, applied to a new series.
2. **Annotated chart** marking each inversion's start date and the following
   recession's start date with `ax.annotate` arrows, so the lead time is visible
   at a glance for each episode.
   *(Revised 2026-09-29: built as a per-episode calendar timeline instead of
   annotations on the spread line. That redesign was accepted in the 2026-09-28
   charts stage but never written back here. On the 1976 sample, only
   **signal** episodes get an arrow to their recession. **Superseded** episodes
   are drawn faded and name the later episode that replaced them. The **open**
   2022 episode gets a dashed arrow to the end of the data.)*
   *(Revised again 2026-09-29, after the false-alarm rule was added. 2022 is now
   a **false alarm**, not open, so its dashed arrow is gone. False alarms (1998,
   2022) get no arrow and are labeled with how long they went without a
   recession. The 2020 continuation is a dotted gray arrow from the 2006
   episode, labeled with the 169 months since the last signal.)*
   *(Revised a third time 2026-09-29, after the symmetric window: 2020 is now
   "no inversion", so the dotted 2006→2020 arrow and its label were removed.
   The 169-month figure moved into the notebook's prose.)*
3. **Bar chart** of "months between inversion onset and recession onset" for each
   **recession**, using the rule under "Assigning a lead time to each
   recession". This is the chart that will visually show whether the lag is
   consistent (supports "reliable") or scattered (supports "noisy").
   *(Revised 2026-09-29: was "for each episode.")* The two false alarms appear
   below the recessions as hatched rows, with a dashed line at the 24-month
   window. The 2020 row is italic text, "no inversion in the 24 months before
   it", with no bar.

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

3. **The same logistic regression with HAC standard errors**
   (`cov_type="HAC", cov_kwds={"maxlags": 12}`). Added 2026-09-28, during the
   tests stage. Tests 1 and 2 as written both assume independent observations,
   which monthly recession data plainly violates. In the 1976-06 sample, the 58
   recession months fall in six contiguous episodes. Re-fitting with Newey-West
   standard errors leaves the coefficient unchanged and asks how much of its
   apparent precision is real. On the original 1990 sample this was the
   decisive result: p moved from 0.0001 to 0.09, so the relationship was not
   significant at 5% once autocorrelation was allowed for. Whether that holds
   with six recessions instead of four is an open question for the redo.

   *(Revised 2026-09-29: this originally said "the 31 recession months are four
   contiguous episodes" and gave the p-values as the finding. Both came from the
   1990 sample.)*

**Limitation to flag up front:** even with six recessions in the 1976-06 to
present window, the regression is best read as a compact description of the
same pattern the charts show, not a rigorously powered causal test. This will
be stated plainly in the conclusion rather than overstating the p-value.
*(Revised 2026-09-29: was "four full recessions in the 1990–present window.")*

## What would support vs. contradict my expectation
- **Supports "noisy":** lead times vary meaningfully across episodes (e.g. some
  6 months, some 18+), and/or the regression coefficient is not statistically
  significant, and/or the 2022–23 inversion behaves differently from the
  earlier ones.
- **Contradicts "noisy" (i.e., signal is reliable):** all episodes precede
  recession within a similar, narrow window, and the regression shows a
  strong, significant relationship.
