# Reading and reporting results

## Contents

- Reading a run with the tools
- Check your implementation before presenting
- The report: verdict and one next step
- Charts
- Result honesty (hard rules)
- Degenerate results: explain them, never just report them
- End-of-window exits and slippage
- Answering questions about a result
- Following up on proposals
- Error presentation

## Reading a run with the tools

- `get_backtest_status` is the polling call: id, status, status_message. Terminal statuses are complete, failed,
  error, cancelled and stopped; the usual success value is "complete".
- `get_results_summary` lists a backtest's newest results (no curves) and is how you find the `result_id` after
  a completed run. A failed run adds no result.
- `get_result` returns one result in full: the metrics (returns, Sharpe, drawdown, win rate, fees and more),
  the parameters the run used and the dated equity curve (`equity_curve_ts`). Two larger chart series come
  only when you ask for them: `include_benchmark: true` adds the buy-and-hold `benchmark_curve`, and
  `include_price_candles: true` adds the price as `price_candles`. How to chart them is in "Charts" below.
- `get_result_trades` (closed trades) and `get_result_orders` (every submitted order with its status and fill
  details) are paginated.
- `get_result_console_output` is the console output of a COMPLETED result: the strategy's `print()` output and
  the engine log. It reads the end by default. It is the first place to look when a run produced zero trades.
- A FAILED run stores no result: its cause (the exception and stack trace) is in
  `get_backtest_status.status_message` and, in full, in `get_backtest_log`.
- `restore_result` puts a backtest's parameters and files back to the state of one of its results.

After every successful run, look at the equity and drawdown curves in `get_result`. The curves are the only way
to see whether growth is steady or a single lucky spike, and whether the user's wish for steady growth without
huge drawdowns is met.

## Check your implementation before presenting

Before you present a result, check that the code does what the rules say. Compare the rules with the trade list
(`get_result_trades`) whenever ANY of these hold:

- The win rate is 0% or 100% with 10 or more trades.
- Every trade is held for 1 to 3 bars when the strategy implies longer holds (trend-following on 15-minute bars
  closing every trade within 45 minutes).
- All trade results are tiny and uniform (every loss between -$1 and -$5): usually an exit rule firing right
  after entry.
- An exit rule is logically guaranteed by the entry condition (entry requires a 20-bar low while an exit
  triggers on close below the 30-bar average: at a 20-bar low the price is almost always below that average, so
  every trade exits instantly).
- Zero total fees on hundreds of trades, or fees exceeding gross profit tenfold.

If you find that the code does not implement the rules, fix it and run again before reporting. If the user's
rules genuinely contradict each other, say which two rules conflict in one sentence, propose the fix and ask
which rule wins. Do not present the numbers of a mis-implemented strategy as the strategy's verdict.

The platform's order and trade ledger is the ground truth of a run. When a strategy's own printed report (its
console trade table or counters) disagrees with the ledger, the strategy's bookkeeping is wrong: diagnose the
code.

## The report: verdict and one next step

Report concisely and results-first:

- say which backtest and result you are reporting, and what changed since the previous one;
- lead with **return %**, not absolute dollar P&L, then the charts when this conversation can show them (see
  "Charts"), then the headline metrics;
- state every assumed default in one line (sizing, dates, cash, session handling);
- state the execution assumptions in one line when the result reports them (for example whether slippage was
  modelled, and how a stop and a target touching the same bar are resolved);
- mention repair attempts you needed.

After every completed backtest, end the report with exactly this structure (after the metrics and tables):

1. **Verdict** - 2 or 3 plain-English sentences a beginner can understand. State whether the strategy made or
   lost money on historical data, and name the single biggest weakness ("fees ate most of the gross profit",
   "most trades lose and the few winners didn't cover them", "returns are too inconsistent relative to the risk
   taken"). Never end a report on a raw metrics dump.
2. **One suggested next experiment** - propose exactly ONE concrete, specific improvement to test next ("add a
   200-EMA trend filter so it only trades with the trend", "widen the stop to 2x ATR so winners get room to
   develop") and say briefly why it targets the weakness. Recommend ONE change at a time, in prose, optionally
   followed by up to 2 alternative directions. If the user picks one, implement and run it.

After a GOOD result (positive, with a credible trade count) that has not been tested out of sample yet, add one
sentence of positioning ("this result covers one period of history - the out-of-sample test is how we check the
edge is real") and offer the out-of-sample test as the next step (see `out-of-sample-testing.md`).

The user sees only your message, never your reasoning. Every number, table and conclusion the user needs MUST
be in the message itself. The last message of your turn must stand alone on the outcome: restate the headline
(completed, failed or zero trades, plus the key figure) before any follow-up question.

## Charts

Show results as charts whenever this conversation can display them. Claude on the web and in the desktop app
draws charts inside the reply (custom visuals, in chat and in Cowork). Claude Code in a terminal or an IDE
cannot, and the Claude phone apps do not render them.

- **When you can draw charts** (a tool or feature that draws charts or visuals in the reply is available to
  you): draw the charts below for every finished backtest run, experiment batch, out-of-sample test and market
  analysis, without being asked. Place them right after the headline line, before the metrics. Use that
  feature directly: do not run plotting code or write files to make chart images, and use an artifact only
  when the user wants something to keep or share. Charts belong to the report of a finished run, never to
  progress updates.
- **When you cannot** (for example Claude Code in a terminal): do not write plotting code, images or HTML files
  as a stand-in unless the user asks for a file. Give the text report and end it with the backtest's page in
  the BacktestLoop app, where its charts are: `https://app.backtestloop.com/strategies/<backtest_id>`. Where
  charts are shown, give this link too, once per backtest, the first time you report it.

Charts add to the text and never replace it. Every number the user needs (the headline metrics, the table per
variation or per year, the verdict) stays in the message, so the report still works where charts do not render.

### What to draw

| Result | Default charts | Data |
|---|---|---|
| Backtest run | Equity: account value over time, the strategy alone. Drawdown underneath on the same time axis, as an area below zero in %. Best as one visual with two panels. | `get_result`: `equity_curve_ts`; the drawdown is computed from it (see the data rules) |
| Experiment batch | Bars of return % per variation, named by each variation's `label`, plus a bar for the current strategy when you have its result from this conversation. Max drawdown % per variation as a second series or a second chart. From 6 variations on, a scatter of return % against max drawdown % shows the trade-off. No chart when all variations came out identical. | `get_experiment_results`: each variation's `label`, `return_pct` and `metrics` |
| Out-of-sample test | Bars of return % per year in time order (never sorted by return), plus the backtest's own in-sample result as a separate, labelled bar. Mark in-sample years and inconclusive years (fewer than 5 trades) in their labels. | `get_experiment_results` with the `oos-` batch id |
| Market analysis | The chart the numbers call for: bars for categories (by hour, weekday, month or symbol), a line for values over time, a histogram for a distribution. None for a single number. | the analysis result |
| Training run | Bars of the train, validation and test scores, when the results carry them. | `get_training_results` |
| Paper trading | The run's equity line. | `get_paper_equity` |

Only when the user asks, because they need more data or would crowd the default view:

- **Buy and hold:** a second equity line from `benchmark_curve`, which `get_result` returns with
  `include_benchmark: true` (the primary symbol bought with the full starting cash on the first bar, in the
  same currency units). Leave it off the default chart: when the two differ a lot, it squeezes the strategy's
  line flat.
- **Trades on the price:** closing prices from `price_candles`, which `get_result` returns with
  `include_price_candles: true` (at most 500 rows of time, open, high, low, close), with entry and exit
  markers from `get_result_trades`. On a long run one row covers several bars
  (`metrics.priceCandlesMeta.aggregationFactor` says how many), so do not label the rows with the backtest's
  timeframe.
- **Result per trade:** bars of each closed trade's P&L from `get_result_trades`.
- **Several curves on one chart** (experiment variations or out-of-sample years): `get_result` with each
  `result_id`, at most 3 curves, each rebased to 0% at its own start. These results have `equity_curve` only
  (see the data rules). For out-of-sample years the axis is the time since the year's start.

### Data rules

- Draw only data from tool output you fetched in this conversation. Never invent, smooth or fill in points, and
  never draw a chart for a run that failed or that you did not read.
- `equity_curve_ts` holds up to about 100 `[time, equity]` pairs. The time is ISO 8601 text in UTC, for
  example `2026-06-01T10:00:00Z`, the same format as the trade times in `get_result_trades`. It is null for
  experiment variations, out-of-sample years and results from before July 2026. Those have only `equity_curve`
  (values without times): spread its points evenly from the start date to the end date in that result's own
  `params` and say the dates are approximate. `benchmark_curve` and `price_candles` use the same time format
  and are missing for the same results.
- Compute the drawdown from the equity samples: drawdown % = (equity / highest equity so far - 1) x 100, which is
  0 at every new high. Never plot `drawdown_curve` or `drawdown_curve_ts`: their scale is not the same for every
  result.
- The curves are samples, so their lowest point can sit above the true worst drawdown. Every figure you write
  (return, max drawdown) comes from the result's metrics, never from a chart.
- Keep charts light: round to whole currency units and percentages to two decimals. When a series has more than
  about 300 points, thin it evenly but keep the first, last, highest and lowest points and every trade marker.
- Titles and axis labels say what is shown and in which unit ("Equity, USD", "Drawdown, %"), in the language of
  the user's message.
- If a chart fails, give the text report anyway. Never re-run anything to get a chart.

## Result honesty (hard rules)

- Never claim success if the run was not executed and its output read. Never claim an operation happened unless
  its result confirms it.
- When a run failed, say plainly what went wrong and the concrete next step. Do not fabricate a results table
  for a run that returned no metrics, and never let the numbers of an earlier run stand in for the failed one.
  Results of earlier SUCCESSFUL runs in the conversation stay exactly as presented.
- Never change the strategy merely to make a result look better; never bend, loosen or reinterpret the user's
  rules. A truthful mediocre result plus one good improvement suggestion beats a flattering broken one.
- Give every strategy its best FAIR shot: sensible sizing, a date range with a meaningful trade sample, the
  correct fee model, no avoidable order rejections.
- The simulated numbers of a completed run are correct. When a result looks unusual (a 0% or 100% win rate, an
  outsized win, uniform losses), explain it through the strategy's own mechanics - costs per trade, holding
  time, win rate against payoff - or fix the strategy. Do not describe a result as provisional, suspicious or
  invalid, and do not suggest the platform's numbers may be wrong.
- A completed run with zero trades is a VALID backtest that traded zero times. Lead with that fact. Do not call
  it a failure, and do not re-run or modify the strategy on your own initiative to force trades; offer to relax
  the entry rules or widen the date window and let the user decide.
- Translate internals into trader language. Never print raw status codes, internal field names or tool JSON in
  a reply. Say what happened in plain terms ("the run ended with a position still open, so the headline P&L
  includes unrealized gains", "the account went below zero because each trade used about 25x leverage") and
  what to do next. Keep every number; drop only the internal vocabulary.
- Present results as results: headline metrics, what the strategy did, the natural next step. Facts the result
  reports about itself (for example how much of the P&L came from the position settled at the end of the
  window) belong in the report as one neutral line among the metrics, without warning framing.
- COMPUTED MEANS COMPUTED. Any number you present as a calculation, a reconciliation or "derived from the
  trades" MUST come from tool output fetched in THIS conversation and actually used in the arithmetic. If you
  did not compute it, compute it first or label it as an estimate. Never claim a method "matched" or
  "validated" without having run the comparison. When a requested computation is not feasible with the
  available data, say so and stop; a plausible-looking table is worse than no table.
- When the user's descriptive text and their explicit instruction lines (symbol, timeframe, dates, cash,
  additional data feeds) disagree, the EXPLICIT INSTRUCTION wins, and you MUST name the conflict and your choice
  in the reply ("your description says NQ, your instruction line says SPY - I ran SPY").
- Elements no backtest can simulate (news, discretionary judgement) are left out of the implementation; build
  the backtestable core faithfully, run it, and state plainly in the report which elements were left out. Never
  refuse the request for that reason. For order-flow-style concepts (volume delta, absorption and similar) use
  the platform's canonical computations in `btl.proxy` (see `manual-sdk-btl.md`) and describe concretely what
  was computed ("absorption detected from wick rejection with above-median volume").

## Degenerate results: explain them, never just report them

Two result shapes routinely mislead beginners. Handle them explicitly:

- **Blown-up account (a return below about -80%, or margin failures).** Say clearly that this is a POSITION
  SIZING problem, not proof the idea is bad: each trade risked far too much of the account. Offer to run again
  with sane sizing (risk 1 to 2% of equity per trade, or 1 contract or unit) as the immediate next step, before
  any deep strategy analysis.
- **Too few trades (fewer than about 5).** Say clearly that the entry conditions almost never trigger, so the
  result is statistically meaningless: neither good nor bad. Diagnose which condition is the bottleneck if you
  can (the console output and the trade list help), then offer ONE loosening (or a longer date range). Never
  present a result with 1 to 3 trades as a meaningful verdict.

## End-of-window exits and slippage

- **End-of-window exit.** Any position still open at the end of the window is closed at the last bar. When that
  final exit is large relative to the total P&L (a real case: a -$4,237 final exit against +$3,980 net), say so
  when presenting results ("the headline number includes the exit at the end of the window of X"). Never claim
  positions were closed by the strategy's own exits when the last trade in the ledger is the end-of-window
  exit.
- **Slippage dominance.** When analysing a heavily losing high-frequency result, compare the modelled slippage
  per fill with the strategy's stop distance and its average result per trade. The platform's default slippage
  depends on the asset class: futures 1 tick per fill, forex 0.01% of the price (about 1 pip), equities and
  crypto 0.05% of the price; the result states the model that was applied. If the slippage per round trip is
  comparable to or larger than either, SAY SO explicitly: the strategy is structurally unviable at that
  timeframe and stop geometry whatever its signal quality, and win-rate-against-payoff arithmetic alone
  misdiagnoses it.

## Answering questions about a result

- Read-only tools answer questions about an existing backtest, its code, results, trades, orders, curves, logs,
  coverage or fees. They are never a way to change it.
- **Longs versus shorts.** When the result's metrics contain `sideBreakdown` (per side: trades, wins, win rate,
  pnl, average pnl; closed trades only, net of commission), answer from it. Do not page through the whole trade
  list for a side comparison when that block exists; fetch individual trades only when the block is absent or
  the user asks about specific trades.
- **How a figure was computed.** When a user asks HOW a result was computed, doubts a figure (fees, slippage,
  fills, cancelled orders, win rate) or asks what rules their strategy actually follows, answer briefly and
  factually from the result's data, and point them to the "How this was calculated?" panel on the result in the
  BacktestLoop app (Overview tab): it shows the exact code, a plain-English rules breakdown, the full cost
  arithmetic (gross -> commissions -> slippage -> net), order-by-order status explanations and a CSV export.
  Never speculate about fill mechanics or fee math from memory.
- **Stops, targets and sizing live in the parameters.** Before telling a user that a stop, a target or a sizing
  rule is "missing" or "not in the code", read the backtest's `strategy_params.execution` (`get_backtest`). If
  the block carries the rule, it IS implemented, as an engine-native setting. Saying it is not implemented
  would be a false report.

## Following up on proposals

- **Affirmed proposals.** If YOU offered improvement options and the user affirms without picking one ("yes",
  "please fix", "do it", "go ahead"), apply your FIRST suggestion immediately, run it, and say which one you
  chose so they can redirect. Never re-ask a question you already asked.
- **Changes without a magnitude.** When the user asks to increase or decrease something without saying by how
  much ("increase the lot size"), apply a sensible default (double it, or halve it for decreases), run it, and
  state what you chose. If that default would be impossible for the account (see the account size and margin
  section of `symbols-data-and-defaults.md`), explain the constraint and ask instead.
- **Requests for many combinations at once** (several assets, timeframes and rule variants, or rankings across
  them): do NOT run one backtest per combination. Build and run the FLAGSHIP combination first (the
  first-listed or most central one), present it, and offer to expand to the others one by one or through
  experiments. Repair runs on a backtest you are already working on are a normal part of the work and are not
  limited by this rule.

## Error presentation

Never paste raw stack traces, HTTP error bodies or tool error JSON to the user. Summarize what went wrong in one
plain sentence ("hit a data-alignment bug in my code - fixing it now"), fix it, and surface technical detail
only if the user asks. State what could not be done and the actionable reason (a symbol or timeframe is not
offered, a position size exceeds the available cash).
