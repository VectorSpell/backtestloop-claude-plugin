# Experiments (running variations without changing the backtest)

When the user wants to TRY variations, compare alternatives, optimize or sweep parameters, or test an idea
without committing to it ("try stop losses of 1/2/3%", "compare faster EMA pairs", "run 4 variations"), use
`run_experiments` instead of editing the backtest and running it. When the user asks to change or fix the
backtest itself, edit its files or parameters and run it as usual. If the intent is unclear, prefer
experiments: they are non-destructive.

**Experiments versus out-of-sample tests.** Experiments answer "which VARIATION of my strategy is better?"
(parameter or rule sweeps on one period). The out-of-sample test answers "does my strategy's EDGE survive other
periods?" (the identical strategy on other calendar years, see `out-of-sample-testing.md`). Route by that
question, never conflate the two, and never rank out-of-sample years against experiment variations.

## Rules

- **Experiments NEVER modify the backtest.** Each variation is the backtest's current state plus your
  overrides. Do NOT edit files or update parameters to set up an experiment; pass every change as an override.
- **Strategy tunables** (periods, thresholds) go in `param_overrides.strategy_params`, for example
  `{"strategy_params": {"fast": 5, "slow": 20}}`. That dict is what `signals(data, params)` receives (and what a
  Backtrader strategy reads through its `params`). It REPLACES the backtest's `strategy_params` wholesale for
  that variation, so send the complete object each time, including an `execution` block the backtest already
  has. This only works when the strategy reads the value (`params.get("fast", 10)`); when writing strategies,
  read tunables from params with defaults so they can be swept.
- **Stops, targets, trailing stops, break-even moves, time stops and sizing** live in
  `strategy_params.execution` as engine-native settings and are ALWAYS sweepable with no code change (for
  example vary `execution.stop_loss.value` across 0.01, 0.02 and 0.03).
- **Values hardcoded in the strategy code.** First check the backtest's `strategy_params.execution`: the value
  may already be a setting. Only when it is truly hardcoded in the code, either pass a full replacement file in
  `file_overrides` for each variation, or (when the user agrees to change the backtest) expose it as a strategy
  parameter the code actually READS, with unchanged behaviour, and then sweep that parameter.
- **Code changes** go in `file_overrides` (FULL file contents, never diffs); code shared by all variations goes
  in `shared_file_overrides` once; files to remove go in `file_deletes`. Override code gets NO syntax
  check before it runs: a runtime error (a missing import, a vectorbt-pro-only API such as `wtype=`) fails that
  variation outright. Every override file must carry the full import preamble and use only the open-source
  vectorbt API (see `code-generation-spec.md`); when in doubt, compute indicators in plain pandas or with `btl`.
- **Top-level `param_overrides` keys** are for real backtest parameters only (symbol, timeframe, start_date,
  end_date, initial_cash, engine and so on).
- Give each variation a short meaningful `label` ("SL 2%", "EMA 50/200").
- Each variation uses credits like a normal run. Only one batch can be active per backtest, and the number of
  variations per batch is limited by the user's plan. Keep batches focused (a handful of variations); for large
  sweeps, run a coarse batch first, then refine around the best result. Filter logically invalid combinations
  (`fast >= slow`) before you propose them.
- Experiments cover variations of ONE backtest. A request spanning several SYMBOLS needs separate backtests:
  build the first-listed or most central one, present it, and offer the rest.

## Running and reading a batch

1. `run_experiments` with `backtest_id` and `variations` (plus `shared_file_overrides` when needed). It returns
   the `batch_id`.
2. Poll `get_experiment_results` (with the `batch_id`; without it you get the latest batch) until the batch
   status is terminal. Each finished variation carries its status, headline metrics and a `result_id`.
3. `list_experiment_batches` lists all batches of a backtest. A backtest with no batches answers
   `get_experiment_results` with status "none".
4. If the user asks to stop, call `cancel_experiments`. If a batch was already running when you were asked to
   start a new one, say so and offer to cancel it instead of starting another.

## Reporting

- Summarize the comparison and name the best variation with brief reasoning (return, drawdown, trade count),
  together with its caveats. Beware variations that "win" on 1 or 2 trades.
- If all variations came out identical, say so plainly. Never present a winner from an identical batch; find
  out why the overrides had no effect (usually a parameter the code does not read).
- If NO variation beats the current strategy, say so honestly.
- Where this conversation can show charts, add bars of return % per variation (and max drawdown % beside them)
  after the headline; see "Charts" in `results-and-reporting.md`. Keep the comparison table in the text too.

## Applying a variation

- **Never apply a variation on your own initiative.** Offer it, and apply it only after the user confirms: it
  replaces the backtest's current code and parameters.
- **A value the user asked for is already confirmed.** When one request asks for a change AND a comparison
  ("add a 5% stop loss and compare stop losses of 3, 5 and 8%"), run the comparison as a batch, then apply the
  variation that carries the requested value (here the 5% one) with `restore_result`, and say that you did.
  Do not ask again, and do not run the backtest a second time for it. Any other variation still needs the
  user's yes.
- "Apply" or "promote" ALWAYS means `restore_result` with that variation's `result_id`. NEVER recreate a
  variation by hand through `update_backtest` or file edits: `restore_result` restores the variation's exact
  snapshot (parameters, code AND files), and a manual copy can silently miss differences between the variation
  and the current strategy. The previous state stays available among the backtest's results, so applying is
  reversible.
- **Applying is NOT a run, and it is final.** The variation already executed against this exact backtest and
  produced a real result; `restore_result` makes that result the backtest's current state. NEVER re-run the
  backtest after applying, and never suggest a re-run "to make it official": an identical re-run only duplicates
  the result and spends the user's credits. Run again only when something CHANGES afterwards.
