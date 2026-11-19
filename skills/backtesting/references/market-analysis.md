# Market analysis

`create_market_analysis` runs your own Python over historical market data without creating a backtest. Use it
for questions ABOUT the market data itself: average daily range, volatility by hour or session, gap behaviour,
correlations between symbols, days with volume spikes, "compare TSLA daily versus hourly volatility this month".
It is NOT a strategy backtest: it runs your code over the data and returns whatever you return. It is also not
for questions about a backtest's results; use the result tools for those.

When a message contains explicit strategy rules (entries, exits, stops) and asks to backtest, build or test
them, build the backtest. An analysis may run IN ADDITION to that, never INSTEAD of it.

## Calling it

- `code`: Python defining `def analyze(data, params)`.
- `sources`: 1 to 4 sources, each `{symbol, timeframe, start_date, end_date, alias?}`. `timeframe` is one of
  1m, 5m, 15m, 1h, 1d. Each (symbol, timeframe) pair must be unique. Each source may have its own date range.
  Set `alias` to control that source's key. Pass tight sources: the smallest date range that answers the
  question.
- `strategy_params` (optional): a free-form dict handed to `analyze()` inside `params`.
- `name` (optional): a short display name.

Check `get_symbol_coverage` for every symbol first. For broader scans ("test this on 12 big stocks"), run
sequential analyses of at most 4 sources and merge the findings yourself.

The analysis is asynchronous: `create_market_analysis` returns its id, `get_market_analysis_status` is the
polling call (it never includes the result), and `get_market_analysis` returns the result once the status is
complete. `list_market_analyses` lists earlier analyses without their results; `cancel_market_analysis` stops a
queued or running one.

## Inside `analyze()`

- `data` is a dict of Polars DataFrames, one per source, keyed by the source's `alias` (or `"SYMBOL_timeframe"`
  when there is no alias, for example `"TSLA_1d"`). There is no "primary" key: iterate
  `for key, df in data.items():` (or read the keys from `params["sources"]`); never guess key names.
- Each frame's columns: `timestamp` (epoch milliseconds, UTC), `open`, `high`, `low`, `close`, `volume`,
  `symbol`, `instrument_id`. polars (`pl`), pandas (`pd`) and numpy (`np`) are already imported in your
  namespace: use them directly (for example `pd.Timestamp(ts, unit="ms")`); no import line is needed.
- The frames are **Polars**, not pandas, and you must not mix the two. **The simplest reliable approach:
  convert once at the top - `pdf = df.to_pandas()` - and do EVERYTHING in pandas** (`pct_change`, `cummax`,
  `idxmax`, `.iloc`, `.rolling` and so on). If you stay in Polars instead, use only Polars APIs (cumulative
  operations are `cum_max`/`cum_min`/`cum_sum`, not `cummax`/`cumsum`) and never feed a pandas or numpy value
  back into a Polars object: indexing a Polars Series with the `numpy.int64` from `idxmax()` raises
  `TypeError: cannot select elements using key of type 'numpy.int64'`. Don't half-convert.
- Keep the code focused and return COMPACT summaries (aggregates, not raw bars): a few kilobytes. The return
  value must be JSON-serializable. A result too large to return comes back from `get_market_analysis` as a
  short preview with `result_too_large` set; run the analysis again returning aggregates.

## Time zone and session contract (hard rule)

A session-shifted analysis once convinced a user that correct data was corrupted. Follow this every time:

- Bar timestamps are UTC, and equity intraday feeds INCLUDE pre-market and post-market (04:00-20:00 Eastern).
- The `timestamp` column is EPOCH MILLISECONDS (Int64). In Polars, parse it with
  `pl.from_epoch(pl.col("timestamp"), time_unit="ms")`, then `dt.replace_time_zone("UTC")`, before converting
  to the exchange time zone.
- POLARS TRAP (proven): `dt.hour()` returns Int8, so `hour()*60+minute()` silently OVERFLOWS into garbage and a
  session filter matches ZERO bars. Always cast first: `dt.hour().cast(pl.Int32)*60 + dt.minute()`.
- If a session filter returns zero bars on data that loaded, check the epoch unit, the time-zone conversion AND
  the integer dtypes of your derived columns FIRST.
- Any analysis touching sessions, opening ranges, or daily aggregation of intraday bars MUST convert timestamps
  to the exchange time zone first (each `params["sources"]` entry carries `exchange_timezone` and
  `session_note`). Filtering "09:30-15:59" on raw UTC timestamps selects pre-market plus the morning and misses
  the afternoon.
- A daily (1d) or weekly bar is stamped 00:00 UTC of its trading day: read its date straight from the UTC
  timestamp. Do NOT convert these timestamps to the exchange time zone and do not filter them by session
  time. Converting moves every bar to the evening before, so Monday's bar reads as Sunday and every
  weekday figure is shifted by one day. The conversion rule above is for intraday bars only.
- Daily (1d) bars are regular-session and unadjusted (comparable to TradingView). Intraday highs and lows
  outside the daily bar's range are extended-hours prints, not data errors.
- NEVER tell a user the platform's data is wrong from a daily-versus-intraday or TradingView comparison until
  you have re-checked with an explicit exchange-time-zone regular-session filter (equities: 09:30-16:00
  America/New_York) and the discrepancy survives.

## Example

```python
def analyze(data, params):
    out = {}
    for key, df in data.items():          # key = alias or "SYMBOL_timeframe"
        if df.height == 0:
            out[key] = {"error": "no data"}
            continue
        pdf = df.to_pandas()              # convert ONCE, then use pandas only
        close = pdf["close"]
        rets = close.pct_change().dropna()
        out[key] = {
            "bars": len(pdf),
            "return_pct": round((close.iloc[-1] / close.iloc[0] - 1) * 100, 2),
            "annualized_vol_pct": round(rets.std() * (252 ** 0.5) * 100, 2),
            "max_drawdown_pct": round((close / close.cummax() - 1).min() * 100, 2),
        }
    return out
```

## Reporting

Answer the user's question from the returned numbers. Every figure you present must come from the analysis
result fetched in this conversation. If the result came back as a preview only, summarize honestly from what is
available and never present the preview as the full data.

When the answer is a set of numbers (by hour, by weekday, over time, a distribution) and this conversation can
show charts, add the chart that fits it; see "Charts" in `results-and-reporting.md`. The numbers stay in the
text.
