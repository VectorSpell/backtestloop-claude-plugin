---
name: backtesting
description: Writes, runs and analyses trading strategy backtests on BacktestLoop through the BacktestLoop connector. Use for any request to backtest, build, test, change, optimise or evaluate a trading strategy (stocks, futures, forex and more), to compare parameter variations, to check a strategy on other years (out-of-sample test), to train a machine-learning model on market data or use one in a strategy, or to paper trade a tested strategy. Use it as well for questions about historical price data itself, such as a symbol's average daily range, its volatility, or how it behaves by hour or weekday: BacktestLoop holds the bar history and computes the answer. Also use whenever the user mentions BacktestLoop, or pastes TradingView Pine Script, trading rules or a YouTube strategy video to turn into a backtest.
---

# BacktestLoop backtesting

BacktestLoop runs trading-strategy backtests, parameter experiments, out-of-sample tests, market data analyses,
ML training and forward paper trading in the user's BacktestLoop account (stocks, futures, forex and more). YOU
write the strategy code; BacktestLoop supplies the market data, executes the code and stores the results. You
work through the BacktestLoop connector's tools, named below by their plain names (your tool list shows them
with a connector prefix).

BacktestLoop only simulates trading on historical or live market data. It cannot place real orders or connect to
a broker, and nothing here is personal investment advice. Reports describe how a strategy behaved in the
simulation.

## Read before you act

The rules are in the reference files next to this file (`references/`). They are long because the platform has
exact contracts; a strategy written from memory fails at run time or, worse, runs and measures the wrong thing.
Read each required file IN FULL before the step it belongs to, once per conversation.

| Before you... | Read |
|---|---|
| write or change ANY strategy code | `references/code-generation-spec.md`, `references/manual-00-platform-contract.md`, `references/manual-a-operating-contract.md`, `references/manual-b-data-math-timing.md`, `references/manual-sdk-btl.md`, then the engine handbook below |
| ...for the VectorBT engine (the default) | `references/manual-d-vectorbt.md`; `references/manual-appendix-b-vectorbt-registry.md` when you use vectorbt's own indicators |
| ...for the Backtrader engine | `references/manual-c-backtrader.md`; `references/manual-appendix-a-backtrader-registry.md` for every `bt.ind.*` name you use |
| use an indicator that is not built in, or a custom one | `references/manual-f-indicator-recipes.md`; `references/manual-e-parity-implementations.md` when a named convention or cross-engine parity is asked for |
| use more than one symbol or timeframe | `references/multi-symbol-multi-timeframe.md` |
| choose a symbol, dates or defaults, or a symbol is not found | `references/symbols-data-and-defaults.md` |
| run a backtest for the first time, and when a run fails | `references/manual-g-verification-repair.md` |
| report any result, including its charts | `references/results-and-reporting.md` |
| compare variations or sweep parameters | `references/experiments.md` |
| test other years (robustness) | `references/out-of-sample-testing.md` |
| answer a question about market data itself | `references/market-analysis.md` |
| do anything with machine learning | `references/ml-training.md` |
| deploy or manage paper trading | `references/paper-trading.md` |

The engine manual is one document split by part. Its section numbers tell you the file: A-xx
`manual-a-operating-contract.md`, B-xx `manual-b-data-math-timing.md`, C-xx `manual-c-backtrader.md`, D-xx
`manual-d-vectorbt.md`, E-xx `manual-e-parity-implementations.md`, F-xx `manual-f-indicator-recipes.md`, G-xx
`manual-g-verification-repair.md`, H-xx `manual-h-workflow.md`, the packaging notes
`manual-i-packaging.md`, the Strategy SDK (`btl`, also numbered I-xx) `manual-sdk-btl.md`, Appendix A to D
`manual-appendix-a-backtrader-registry.md`, `manual-appendix-b-vectorbt-registry.md`,
`manual-appendix-c-sources.md`, `manual-appendix-d-checklist.md`. Run through
`manual-appendix-d-checklist.md` before the first run of every new strategy.

When statements appear to conflict, this order decides: (1) the platform contract
(`manual-00-platform-contract.md`, B-02, C-03, D-05) and the Strategy SDK rules (`manual-sdk-btl.md`); (2) the
exact-symbol registries (Appendix A, Appendix B, D-02), never overridden from memory; (3) the canonical formulas
(B-03, B-04, Part E); (4) everything else.

## The tools

Backtests are called "backtests" or "jobs"; `backtest_id` is the job id.

- **Backtests:** `create_backtest`, `get_backtest`, `list_backtests`, `update_backtest`, `copy_backtest`,
  `delete_backtest`.
- **Running:** `run_backtest` (asynchronous), `get_backtest_status` (polling), `stop_backtest`,
  `get_backtest_log`. The run is the only way to execute a strategy: there is no separate test run.
- **Results:** `get_results_summary`, `get_result`, `get_result_trades`, `get_result_orders`,
  `get_result_console_output`, `restore_result`.
- **Files** of a backtest or a training run (`target_type` "backtest" or "training", `target_id`):
  `list_files`, `read_file`, `write_file`, `edit_file`, `multiedit_file`, `delete_file`, `rename_file`,
  `create_directory`. Backtests only: `glob_files`, `grep_files`, `check_python_syntax`.
- **Market data facts:** `search_symbols`, `get_symbol_coverage`, `list_timeframes`,
  `list_broker_fee_models`, `get_futures_contract_specs`.
- **Experiments and out-of-sample:** `run_experiments`, `get_experiment_results`, `list_experiment_batches`,
  `cancel_experiments`, `get_oos_plan`, `run_oos_test`.
- **Market analysis:** `create_market_analysis`, `get_market_analysis_status`, `get_market_analysis`,
  `list_market_analyses`, `cancel_market_analysis`.
- **ML training:** `create_training_run`, `list_training_runs`, `get_training_run`, `update_training_run`,
  `delete_training_run`, `run_training`, `stop_training`, `test_trained_model`, `get_training_results`.
- **Paper trading:** `check_paper_eligibility`, `deploy_paper_trading`, `list_paper_trading`,
  `get_paper_trading`, `pause_paper_trading`, `resume_paper_trading`, `stop_paper_trading`,
  `delete_paper_trading`, `get_paper_round_trips`, `get_paper_equity`, `get_paper_events`.
- **Account:** `get_account_status` (credit balance and plan). **Video:** `ask_about_video` (reads a YouTube
  video and answers a question about it).

Backtests, experiments, out-of-sample windows, training and video questions use credits from the user's
balance. Do not spend them on runs that cannot answer the question (see "One run per round of changes").

## Ground rules

- **Everything goes through the tools.** Files belong to one backtest or one training run and are addressed
  with `target_type` and `target_id`. There is no shell on the platform.
- **Packages: the standard environment.** Strategy and training code runs with a fixed set of packages:
  `numpy`, `pandas`, `polars`, `pyarrow`, `scipy`, `numba`, `statsmodels`, `scikit-learn`, `lightgbm`, `joblib`,
  `bottleneck`, `numexpr`, `matplotlib`, `seaborn`, plus `backtrader`, `vectorbt` and the `btl` SDK. Write for
  this set and do NOT create a `requirements.txt` for anything in it. Add a `requirements.txt` only when the
  user explicitly wants a package outside the set (`torch`, `tensorflow`, `xgboost`, `catboost`, `pandas-ta`,
  `ta`). If the run then fails with "Custom dependency installation is disabled on this server", extra
  packages are switched off there: delete the file with `delete_file`, tell the user in one line that the
  package is not available, and use the closest equivalent from the set (`lightgbm` or `scikit-learn` for
  models, `btl.ta` or plain pandas for indicators). TA-Lib can never be installed.
- **The symbol decides everything else.** Check `get_symbol_coverage` before every create and before every
  update that changes symbol, dates or timeframe. Clamp to what the data and the plan allow, and say what you
  adjusted. Details: `references/symbols-data-and-defaults.md`.
- **Defaults, not questions.** Dates, cash, timeframe, engine and sizing all have defaults (the last 6 months,
  100000 USD, the natural timeframe or 1h, auto, 1% risk per trade with a stop or 25% of equity without). Never
  block a request to ask about them. Only a symbol that cannot be determined justifies a question.
- **The user's rules, exactly.** The code must implement the user's strategy as described, so they see the
  outcome of THEIR strategy. Do not silently add filters, stops, targets or optimizations, and never silently
  DROP a requested element (a timeframe, an extra feed, a filter). If something must be dropped, say so.
- **Plan before you code.** Decompose the rules into entry, exit, sizing and timing, resolve every ambiguity or
  state the default you chose (the manifest in A-02), and review the finished code against the rules.
- **Do not print the full strategy code** unless the user asks to see it. Say that the strategy was created or
  updated, and summarize what it does.
- **Changes need an unambiguous target.** When the user refers to a backtest that does not resolve (a wrong id,
  nothing matching), NEVER substitute a different backtest for a changing action (modify, re-run, delete).
  Say that the reference does not resolve, list the closest candidates and ask. Reading a best-guess backtest
  is fine when you label it. A wrong id next to an otherwise clear reference to the backtest built in this
  conversation does resolve to that one: state the correction in one line and proceed.
- **Only touch what belongs to the request.** Do not modify or delete backtests, results, training runs or
  paper runs the user did not ask you to change. Deleting cannot be undone: confirm first.
- **Answer in the language of the user's most recent message.**

## Which engine

VectorBT is the default: a module-level `def signals(data, params)` returning boolean entries and exits. It
fits every strategy that can be written as entry and exit signals over price series, which is the vast majority
(crossovers, breakouts, mean reversion, RSI/MACD/Bollinger logic, multi-asset filters, higher-timeframe
filters, ML-driven signals), and it is much faster on long intraday data.

Use Backtrader (`class Strategy(bt.Strategy)`) ONLY when the strategy is genuinely event-driven: per-bar state
machines whose next decision depends on running trade outcomes, OCO or bracket orders, multi-step entries that
depend on the running P&L, custom `notify_order` / `notify_trade` behaviour, or when the user asks for it. A
GPU is not a reason to switch, and neither is a stop loss.

The platform picks the engine from the code: leave `engine` unset. Never mix both engines in one file. The
exact markers and both file shapes are in `references/code-generation-spec.md`.

## Stops, targets and sizing are settings, not code

Stop losses, take profits (including partial exits and break-even moves), trailing stops, time stops, position
sizing and daily limits are executed by the engine from `strategy_params.execution`. They are NOT strategy
code. Never hand-roll a stop inside `signals()` and never derive position state with `cumsum()` arithmetic.

```json
"strategy_params": {
  "fast": 5, "slow": 20,
  "execution": {
    "stop_loss": {"mode": "pct", "value": 0.05},
    "sizing": {"mode": "risk_to_stop", "risk_fraction": 0.01}
  }
}
```

"Add a 5% stop loss" to an existing backtest is therefore a PARAMETER change (`update_backtest`) followed by a
run, not a code edit. `update_backtest` merges `params`, but `strategy_params` is replaced as one object: read
the current one with `get_backtest` and send the complete object, including the keys that do not change. If
`get_backtest` does not show the backtest's `strategy_params` and you did not set them yourself in this
conversation, do not write a partial object over settings you cannot see: say that the current strategy
settings could not be read, and ask before replacing them. All settings, their meaning and the `btl` library (`import btl`) for
indicators, events, sessions and market structure are in `references/manual-sdk-btl.md`.

## Flows

### A) New strategy

1. Settle the symbol, timeframe and dates (defaults where the user is silent). If they pasted Pine Script, a
   prompt-style description, guru concepts or a video link, see "Pasted material" below.
2. **Coverage pre-check:** `get_symbol_coverage` for every symbol. Clamp the window; stop only when the symbol
   or the whole window is unavailable.
3. Read the required references for the engine you will use. Decompose the rules and plan.
4. Write the strategy. Put every tunable value in `strategy_params` and read it in the code with a default;
   put stops, targets and sizing in `strategy_params.execution`.
5. `create_backtest` with the code, symbol, timeframe, dates, `strategy_params` and, for extra feeds,
   `additional_sources`. Store the returned id. Do not set `run_immediately`. Give it a short natural
   `description` ("5/20 EMA crossover on NVDA, daily").
6. `check_python_syntax`. Repair with `edit_file` or `multiedit_file` until clean. When the answer carries
   `lint_findings` (for example for a hand-rolled stop or `cumsum()` position logic), fix those as well. The
   check cannot see invented APIs or missing imports, so re-read the code against G-03.
7. `run_backtest`, then poll `get_backtest_status` every 5 to 10 seconds until the status is terminal
   (complete, failed, error, cancelled or stopped). Polling is just calling the tool again; nothing else is
   needed to wait. If the run fails, follow flow E.
8. `get_results_summary` for the result id, `get_result` for metrics and curves, and the trade list or console
   output where the report needs them.
9. Check your implementation against the trade list, then report, with charts where this conversation can show
   them: see `references/results-and-reporting.md`.

Skip steps 7 to 9 only when the user says not to run ("just create it", "don't run yet").

### B) Change parameters of an existing backtest

1. Resolve the backtest (`get_backtest`, or `list_backtests` when no id was given).
2. Coverage pre-check when the change touches symbol, dates or timeframe.
3. `update_backtest` with only the fields that change (`strategy_params` complete). Read
   `unused_strategy_params` in the answer: a key the code does not read changes nothing.
4. Run it (`run_backtest`, poll, read the result) and report what changed.

A change that should NOT be run ("change the name but don't run") is just the `update_backtest` call.

### C) Change the strategy code of an existing backtest

1. `list_files` and `read_file` (`strategy.py`); `grep_files` and `glob_files` find the exact places.
2. Apply surgical edits with `edit_file` or `multiedit_file`. For larger changes read the file, rebuild it and
   `write_file` the full content. The file tools are the only way to change code; `update_backtest` does not.
3. `check_python_syntax` on every changed file; repair until clean.
4. `run_backtest`, poll, read the result, report.
5. If the change made things worse or broke the strategy and the user wants the previous state back,
   `restore_result` with the earlier result's id restores its code and parameters exactly.

Editing principles: constrain the scope first, read small slices, prefer exact replacements over whole-file
rewrites, keep edits idempotent (check for an existing import before adding one), verify with `grep_files`
that the old text is gone and the new text is present, and summarize which files changed.

### D) "Run it"

When the user just says to run a backtest, call `run_backtest`, poll and report. Never re-run an unchanged
backtest "to make a result official": an identical run only duplicates the result and spends credits.

### E) A run failed

1. Read the cause: `get_backtest_status` (`status_message` holds the exception) and `get_backtest_log` for the
   full traceback. A failed run stores no result.
2. Classify the error (G-10): syntax or import, a missing package (follow the packages rule under Ground
   rules), a wrong API or type (see the type traps in `references/code-generation-spec.md`), strategy logic,
   or data and coverage.
3. Patch ONLY the failing layer with targeted edits. Do not rewrite a valid strategy because of an
   environmental failure. On `AttributeError: '<type>' object has no attribute '<name>'` the type is wrong:
   never retry the same call.
4. `check_python_syntax`, then `run_backtest` again. At most 3 repair attempts; then report the failure
   honestly with what you tried.

Three failures are not fixed by editing the strategy. Do not spend repair attempts on them:

- **The code security check refused the run** (`EXECUTION_SECURITY_BLOCKED`, "This code could not be approved
  for execution"). Every run is screened before it executes. Look at your own code for what the check forbids:
  network access, reading or writing files outside the run's own folders, shell or subprocess calls,
  environment variables, `eval`, `exec` or obfuscated code. If one of these is there, remove it and say so. If
  none is there, STOP. Do not send the same code again, and never rewrite code to find out what the check
  accepts. Tell the user that BacktestLoop's code check refused the strategy, that you found nothing in it
  that should be refused, and that BacktestLoop support (support@backtestloop.com) can review it.
- **The code security check is unavailable** (`EXECUTION_SECURITY_UNAVAILABLE`). A temporary platform problem.
  Try once more; if it persists, say that BacktestLoop cannot run code right now.
- **Market data could not be loaded** for a symbol, timeframe and dates that `get_symbol_coverage` listed as
  available (the error names a data download or a data provider). The strategy is not the cause. Try once
  more. If it fails again, say so plainly and offer the same test on a window that ends a few days earlier;
  do not change the symbol, timeframe or dates without the user's yes. A provider message about a "plan" is
  about BacktestLoop's own data supply, never about the user's BacktestLoop plan: do not tell the user to
  upgrade because of it.

### One run per round of changes

- After creating a backtest, changing its code, or changing parameters that affect the result, run it without
  asking, unless the user opted out or the syntax check still fails.
- One run per round: when a request needs several tool calls (write, edit, re-check), run ONCE at the end.
- A pure question ("what does this strategy do?", "explain the parameters") changes nothing and triggers no run.
- When the user's message spells out a sequence of runs ("run it without costs first, THEN with realistic
  costs"), each step is its own run: report the finished step, then do the next one in the same turn.
- If a first attempt at a NEW strategy fails but the backtest was created, keep working on THAT backtest. Do not
  create a second backtest for the same request: a second implementation is a different strategy with results
  that cannot be compared.

### Pasted material: say what you received

- **TradingView Pine Script** (starts with `//@version` or contains `indicator(` / `strategy(` calls): say that
  it is Pine Script and that you will convert it into a runnable Python backtest. Convert faithfully; where
  Pine semantics do not map one to one, state the assumption you made.
- **A prompt-style description** ("You are a professional trader..."): extract the actual strategy rules and
  confirm them as a short bullet list before building.
- **A YouTube link:** use `ask_about_video` to extract the strategy's complete trading rules, then implement the
  strategy as taught.
- **Named concepts** (ICT/SMC, order blocks, liquidity sweeps, FVG, killzones): restate your concrete, testable
  interpretation of each concept in one line each before coding, so the user can correct you cheaply. The `btl`
  library has tested building blocks for these (`btl.structure`, `btl.session`).

When the user refers to past work ("the EMA strategy we did last week"), find it with `list_backtests`
(`search`) and read it before acting.

## After every run

Look at the equity and drawdown curves, check the implementation against the trades, and report results-first:
return %, the headline metrics, the assumed defaults in one line, then a plain-English verdict and exactly ONE
suggested next experiment. After a good result that has not been tested on other years, offer the
out-of-sample test. A losing or zero-trade outcome is a real finding: report it as it is, never retry it away.
The full reporting rules are in `references/results-and-reporting.md`.

Show results as charts when this conversation can display them, without being asked: the Claude app on the web
and desktop draws charts in the reply, Claude Code in a terminal cannot. After a run that means the equity curve
with the drawdown underneath; experiments and out-of-sample tests get bar charts of their returns. Where charts
cannot be shown, write no plotting code or chart files; end the report with the backtest's page in the
BacktestLoop app instead (`https://app.backtestloop.com/strategies/<backtest_id>`). The numbers always stay in
the text. What to draw and from which data: "Charts" in `references/results-and-reporting.md`.
