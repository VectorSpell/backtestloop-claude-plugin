# Paper trading

Paper trading forward-tests a COMPLETED backtest result on real, live market data with VIRTUAL funds. It is
never real money, never a broker, and never live trading with real execution: BacktestLoop cannot place real
orders. Use it when the user wants to run a strategy "live", "deploy" it, "paper trade" it, or watch it on real
market data.

Facts you may state:

- Paper trading is part of the paid plans. The number of strategies that can run at the same time (slots)
  depends on the plan.
- It consumes no credits.
- A paused run can be resumed. A COMPLETED run is final and can never resume; start a fresh run from the same
  result instead.
- At most one resumable run exists per result. Completed runs are history and never block a fresh start.

## Tools

| Action | Tool |
|--------|------|
| Could this result be deployed right now? | `check_paper_eligibility` (`backtest_id`, `result_id`) |
| Deploy | `deploy_paper_trading` (`backtest_id`, `result_id`, optional `name`, `starting_cash`, `timezone`, `email_mode`) |
| List runs and plan slot usage | `list_paper_trading` (`include`: all, active or completed) |
| One run: status, equity, cash, open position, realized P&L, stats | `get_paper_trading` |
| Pause (reversible, frees the slot; optional `flatten`) | `pause_paper_trading` |
| Resume a paused run (needs a free slot) | `resume_paper_trading` |
| Complete a run (final; flattens the open virtual position by default) | `stop_paper_trading` |
| Delete a run that is not running, with its history | `delete_paper_trading` |
| Closed round trips (P&L, fees, MAE/MFE, R multiple) | `get_paper_round_trips` |
| Equity points | `get_paper_equity` |
| The run's event journal (fills, pauses, position changes) | `get_paper_events` |

## Deploying

`deploy_paper_trading` is ONE call: the eligibility check, a safety rehearsal and the start all happen on the
platform, which takes about 30 to 90 seconds. Pass the `backtest_id` AND the `result_id` of a COMPLETE backtest
result (`get_results_summary` gives the result ids). It always returns a status you must act on:

- **deployed** - report the run. Say what to expect honestly: decisions happen at bar close on the strategy's
  timeframe, equities sleep outside market hours while crypto runs around the clock, and a strategy with a
  regime filter may correctly stay flat for days. Email alerts are on by default (`email_mode` "smart"; the
  other modes are hourly, daily, operational and off); delivery, quiet hours and webhooks are configured on the
  run's page in the BacktestLoop app. If the result has not been tested out of sample yet, add ONE sentence
  recommending that test. It is a recommendation, never a blocker.
- **already_running** - this result ALREADY has its one resumable paper run. Point to that run; never retry the
  deploy.
- **refused** - read the reasons and explain them plainly (for example the plan does not include paper trading,
  or all of the plan's slots are in use). Refused is a normal outcome to explain, not an error. Paper access
  and slots come from the plan itself; buying credits does not change them.
- **rehearsal_failed** - explain the rehearsal error; it is retryable and consumed nothing.
  SPECIAL CASE: if the error mentions `live_resume_replay_safe`, this is a one-line strategy fix you make
  yourself. Verify that the strategy's internal state rebuilds from replaying recent bars (indicator-driven
  logic does; only counters that span restarts or running-P&L gates do not), add
  `live_resume_replay_safe = True` as the first line of the `Strategy` class body (`edit_file`), run the
  backtest again, and deploy the NEW result it produces. If the state is genuinely path-dependent, implement
  `live_resume_state()` and `load_live_resume_state(state)` instead (see `manual-c-backtrader.md`). VectorBT
  strategies need nothing: they are replay-safe by construction.

Use `check_paper_eligibility` for "could I...?" questions without deploying.

## Managing runs

- Pause, resume, stop and delete a run ONLY on the user's explicit request.
- `stop_paper_trading` completes the run: a terminal stop that freezes the final stats. Stopping takes a few
  seconds to settle; `delete_paper_trading` is refused while the run is still stopping, so read
  `get_paper_trading` until the status has settled before deleting.
- `get_paper_trading` reads the platform's stored copy of the run, which is about 20 seconds behind the engine.
- When reporting a young run's numbers, note that live statistics need time to become meaningful. Never rank a
  days-old paper run against its multi-month backtest.
