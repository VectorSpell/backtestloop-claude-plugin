# Out-of-sample testing (does the edge survive other years?)

A single backtest only proves the strategy fit ONE historical window. `run_oos_test` re-runs the backtest's
CURRENT strategy (same code, same parameters) on individual CALENDAR YEARS. It is the platform's most important
robustness check, and deliberately simple: one row per year.

## When

- When the user asks to validate, stress-test or check robustness, or asks "would this work in other years?".
- After any good full-run result that has not been tested out of sample yet, offer it as the next step.
- It is a different feature from experiments: the strategy stays IDENTICAL and only the period changes (see
  `experiments.md` for the distinction).

## How

1. `get_oos_plan` (read-only) shows the candidate calendar years for a backtest: each year with its data
   coverage and an in-sample flag, the order a bare run would pick them in, and the plan's window limit. Use it
   to pick sensible years when the user didn't name any, and to see whether any out-of-sample year exists at
   all.
2. `run_oos_test` with just `backtest_id` is the normal call. The picker is aware of the backtest: it selects
   years OUTSIDE the backtest's own start and end dates (on BOTH sides of the range, closest years first) and
   never picks years fully inside it. `year_count` controls how many (default 5).
   - Pass `years` explicitly only when the user names them ("test it on 2020" -> `years: [2020]`) or when
     deliberately steering ("test the 2020 crash").
   - `window_years` (1 to 3) sets the window length for low-frequency strategies that trade too rarely for
     single years; `years` are then the START years of the windows.
   - `ranges` (`[{start_date, end_date, label}]`) exists for periods that are not calendar years, but years are
     the normal interface.
3. The test runs as an experiment batch whose id starts with `oos-`. Poll `get_experiment_results` with that
   `batch_id` until the batch is terminal. Each window carries the platform's verdict (whether it is counted,
   and why) and an `in_sample` value ("no", "partial" or "full") against the backtest's own range; the response
   also includes the backtest's own in-sample result as the reference.
4. One batch can be active per backtest: if an experiment or another out-of-sample run is already in flight,
   the call is refused. Say so instead of retrying.
5. The backtest itself is not modified. Each window uses credits.

A year with partial data coverage runs on what exists and is labelled "(partial)". If the plan says no
out-of-sample year exists (the backtest's range already covers all available history), tell the user that
plainly.

## Reporting

- A per-year table (year, return, Sharpe, win rate, trades), then: "profitable in X of N out-of-sample years;
  worst: <year> (<return>)".
- Where this conversation can show charts, add bars of return % per year in time order, with the in-sample
  result as a separate labelled bar; see "Charts" in `results-and-reporting.md`. The table stays in the text.
- Count only years OUTSIDE the backtest's own range in that verdict. Mark partial and full in-sample rows as
  such and never present them as out-of-sample validation.
- Years with fewer than 5 trades are INCONCLUSIVE: say so instead of counting them as pass or fail.
- Report each year's survival plainly and absolutely. Never average years together, and never rank years
  against each other by raw PnL.
- An edge dying out of sample is a finding, not a failure. If the in-sample years look clearly best, tell the
  user plainly to expect live results closer to the other years.
- NEVER apply an out-of-sample year's result to the backtest with `restore_result`: it would overwrite the
  backtest's date range.
- Natural next steps to offer after the report: look into why the worst year failed, add a regime filter and
  re-test, test more years.
- In the BacktestLoop app, the Out-of-Sample Lab on the result page shows the per-year cards, the equity curves
  and an overlay of all tested years. Mention it once.
