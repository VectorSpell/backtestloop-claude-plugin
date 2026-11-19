# Symbols, data coverage and defaults

## Contents

- Supported asset classes and timeframes
- Coverage pre-check (mandatory)
- Reading the coverage answer
- Clamping the window and disclosing it
- Symbol resolution protocol
- Index-like requests
- Known data facts
- Time zones and sessions
- Defaults when the user is silent
- Account size, cash and margin sanity
- GPU

## Supported asset classes and timeframes

Asset classes supported: **equity, futures, crypto, and forex**.

Valid timeframes (the **only** timeframes you may use; refuse anything else):
`1s`, `5s`, `10s`, `30s`, `1m`, `2m`, `3m`, `5m`, `10m`, `15m`, `30m`, `1h`, `2h`, `4h`, `6h`, `8h`, `12h`, `1d`, `1w`.
`list_timeframes` returns the authoritative list.

How much history a user may use depends on their plan and on the timeframe (1-minute and 1-second data have
shorter windows than 5-minute and above). Never guess these limits: `get_symbol_coverage` returns them.

## Coverage pre-check (mandatory)

**Always call `get_symbol_coverage` before:**

- `create_backtest` - for every symbol the backtest will use (the primary one and every additional source).
- `update_backtest` - whenever the update changes the **symbol**, the **start or end date**, or the **timeframe**.
- `create_training_run` and `create_market_analysis` - for every symbol they load.

**Validate the requested configuration** against the answer:

- The requested symbol must exist and support the requested timeframe.
- The requested date range must fall **within the data coverage window** for that symbol (its first and last
  date with data).
- The requested date range must also fall **within the user's plan limits**. The answer gives the longest window
  in days overall and per timeframe, plus ready-to-copy earliest start dates for a window that ends today.

**Re-check coverage** any time the user pivots to a new symbol, timeframe, or date range mid-conversation. Do not
rely on stale coverage from earlier turns. Treat the `get_symbol_coverage` answer as **authoritative**. Do not
guess plan limits or symbol availability.

## Reading the coverage answer

- `exists: true` - the symbol is supported.
- `exists: false` - the symbol is not available under that name. If a `hint` field is present, retry once with
  that ticker immediately. Otherwise follow the symbol resolution protocol below.
- `exists: null` - a lookup error. Treat it as UNKNOWN, retry once, and never start a backtest on a symbol
  whose existence is unknown.
- A `resolution` block (or `resolved_from_alias`) means the requested name is being served by a DIFFERENT
  instrument, possibly on a completely different price scale (for example an index name served through an ETF
  that trades near $45, not near 24,000 index points). BEFORE writing or changing any strategy for that symbol,
  state the substitution to the user in one plain sentence using the resolved symbol and its approximate recent
  price, and remind them that any absolute price levels in their rules (entries, stops, targets, thresholds)
  must be on the resolved instrument's scale. Then proceed in the same turn; this is a disclosure, not a
  blocking question. It applies to EVERY aliased symbol, never only indices.

## Clamping the window and disclosing it

- **Clamp both ends on the first attempt.** Coverage ends earlier than your end date: use the coverage end.
  Coverage starts later than your start date: use the coverage start (a micro futures contract listed in 2019
  cannot honor a 2010 start). Never send a window you already know exceeds the available data or the plan.
- Preserve the requested span length by shifting the start earlier only when that stays within the plan's
  limits AND the coverage.
- **Act, don't block.** When the ONLY problem is that the requested history exceeds what the plan or the data
  allows (5 years asked, about 2 years of 1-minute data available), run the MAXIMUM available window, disclose
  the clamp prominently, and name the alternatives (a coarser timeframe for deeper history, or a plan with a
  longer window). The exception is an explicit instruction to stop ("if the full window is unavailable, STOP and
  report"): honor that literally.
- If the symbol and timeframe combination is unsupported, or the plan blocks the entire requested window, stop
  and tell the user instead of sending a create or update that cannot work.
- **Date ranges in the report.** Whenever the start or end date you ran differs from what the user asked for,
  for ANY reason (data floor, plan limit, weekend), the results reply MUST state the requested range, the range
  actually run, and why, sized honestly ("11 years of the requested history are not available for this symbol").
  Presenting a shortened window as if it were the request is a violation. A one-line mention is enough only for
  trivial trims under about 10%.

## Symbol resolution protocol

When a requested symbol isn't found: call `search_symbols` (fuzzy search) FIRST. Make at most 3
`get_symbol_coverage` probes in total; never brute-force ticker variations. If nothing suitable exists, tell the
user in your next message: (1) which supported instrument is closest and the exact difference (for example "Dow
futures (YM on CME) track the same index and are the standard backtesting proxy for US30"), and (2) ask before
you substitute an instrument yourself.

State the path, not the gap: say what IS supported and move forward ("DXY trades here through its futures
contract DX", "the closest supported instruments are X and Y - I'll use X").

Common mappings: US30/DJ30 -> YM, NAS100/US100 -> NQ, SPX500/US500 -> ES, GOLD -> XAUUSD (spot) or GC (futures).
Indian markets (NIFTY, BANKNIFTY) are not offered: say so immediately, and do not present US instruments as if
they were equivalent for an India-focused trader; offer them only as an explicit "different market, same
mechanics" option.

When the user lists MULTIPLE candidate symbols or contract variants of the same market ("MNQ and NQ"), the
symbol IS determinable: pick the most sensible one (first-listed or most standard), run, and state your choice.

## Index-like requests

When a user says "S&P 500", "Nasdaq", "Dow" and so on with futures-flavored context (sessions, ticks, contracts,
prop firms, "futures"), resolve to the FUTURES contract: micro (MES/MNQ/MYM) for accounts under about $25k,
e-mini (ES/NQ/YM) above, and say which you picked. NEVER silently substitute an ETF (SPY/QQQ/DIA) for a futures
request; offer the ETF only when the user asks for stocks, ETFs or cash-index behavior explicitly.

## Known data facts

- **Futures symbols are continuous.** Do not add ".c.0" (as in "ES.c.0"); use the plain root (`ES`, `NQ`, `GC`).
  `get_futures_contract_specs` returns a root's multiplier, tick size, margin and contract months.
- **Crypto and forex symbols** use the plain pair ticker exactly as `search_symbols` or `get_symbol_coverage`
  returns it. No slashes, dashes, exchange prefixes or suffixes.
- **MGC ticker collision.** The ticker MGC resolves to the Vanguard Mega-Cap ETF (equity), NOT COMEX Micro Gold
  futures. When a user asks for MGC or micro gold, use GC (COMEX Gold futures) and say in one line that gold
  futures run on the GC series here; never present MGC data as gold.
- **Gold.** Spot gold XAUUSD is a supported forex feed and GC is the supported gold future. For a generic "gold"
  strategy either is valid: pick one and say why in one line.
- **Costs are platform-owned.** Commission, margin and contract multipliers come from the broker fee model that
  is resolved automatically for the symbol's asset class. `list_broker_fee_models` shows them; look only when
  the user asks about costs or wants a specific broker profile.

## Time zones and sessions

- Bar timestamps are **UTC**. Equity intraday feeds INCLUDE pre-market and post-market (04:00-20:00 Eastern).
- Daily (1d) bars are regular-session and unadjusted (comparable to TradingView). Intraday highs and lows outside
  the daily bar's range are extended-hours prints, not data errors.
- A daily (1d) or weekly bar is stamped 00:00 UTC of its trading day: its date is the UTC date of the timestamp.
  Never convert a daily bar's timestamp to the exchange time zone; that moves it to the evening before.
- Any rule about sessions, opening ranges or "the 9:30 candle" must convert to the exchange time zone first. In
  strategy code use the `btl.session` helpers (see `manual-sdk-btl.md`), which handle time zones and daylight
  saving. Use America/New_York for US market sessions unless the user says otherwise.
- NEVER tell a user the platform's data is wrong from a daily-versus-intraday or TradingView comparison until
  you have re-checked with an explicit exchange-time-zone regular-session filter (equities: 09:30-16:00
  America/New_York) and the discrepancy survives.

## Defaults when the user is silent

Dates, cash, engine, GPU and timeframe all have defaults. NEVER block a request to ask about them.

- **Dates:** the last 6 months (inside the coverage and the plan's limits).
- **Starting cash:** 100000. Accounts run in USD: when a request states amounts in another currency, say once,
  up front, that you are treating them as the same number of USD ("EUR 10 -> $10"), offer to convert at the
  current rate instead if they prefer, and do not raise the currency again on later results.
- **Timeframe:** infer the most natural one from the strategy, else `1h`.
- **Engine:** leave `engine` unset (auto-detection from the code). GPU: off.
- **Position sizing:** risk 1% of equity per trade when the strategy has a stop loss, otherwise 25% of equity
  per position, set through `strategy_params.execution.sizing` (see `manual-sdk-btl.md`).
- **Intraday equities and futures:** flatten at the end of the session (`"session_flatten": true` in the
  execution settings).
- **Direction and stacking:** long-only and no pyramiding unless the rules say otherwise.

The ONLY thing that may justify a clarifying question is the SYMBOL, and only when it is genuinely
undeterminable from the request. Ask for the symbol alone (one short question, nothing else bundled) and run
the moment the user supplies it. When unsure whether a symbol is supported, call `get_symbol_coverage` FIRST
instead of asking the user or guessing.

State every assumed default in one line when you present results, and attribute it to the default, not to the
user ("default sizing of 25% of equity per position - say the word to change it").

**Meaningful sample size.** When you choose the date range, prefer one likely to produce at least about 30
closed trades for the strategy's signal frequency and timeframe. If a user-specified range yields fewer than
about 30 trades, present the result but lead with that fact and offer the same strategy over the longest range
the plan allows. Never attach a quality verdict (good or bad strategy) to a result with fewer than 10 trades.

## Account size, cash and margin sanity

Do the arithmetic first, every time, before you run a change:

- Sizing beyond about 100% of equity in a cash account, or contracts whose notional or margin exceeds the
  account (5 NQ contracts are about $2.5M notional against $100k cash) cannot be executed as asked. Running them
  spends the user's credits on a number that does not answer their question.
- If a requested change can have no real effect without margin or leverage, say so BEFORE running and ask which
  way to proceed (use leverage where the instrument supports it, raise the cash, or accept the cash-capped
  version). If you noticed the margin issue earlier in the conversation, act on it consistently.
- The simulated broker, like a real one, fills what the account can fund. That is normal execution: present the
  results as the strategy's results, and explain position sizes or margin only when the user asks.

## GPU

Set `gpu_mode` and `gpu_type` ONLY when the user asks or the strategy genuinely needs it (ML inference, heavy
computation); the default is CPU. VectorBT strategies run fine with a GPU: the vectorized portfolio loop runs on
CPU, while ML inference or feature engineering inside `signals()` can use the GPU, so a GPU is never a reason to
switch to Backtrader. GPU access and the available GPU types depend on the user's plan (`get_account_status`).
When the plan has no GPU, do not attempt the GPU run and report a failure: run the SAME work on CPU, and say in
one line that you did.
