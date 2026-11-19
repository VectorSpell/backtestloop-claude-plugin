# Multi-symbol and multi-timeframe strategies

## Contents

- Requesting the extra feeds
- Feed naming and access in both engines
- VectorBT multi-feed example (start here)
- Backtrader multi-feed patterns (feed naming, core rules, copyable examples A to D)
- Backtrader-specific generation rules

## Requesting the extra feeds

Extra feeds are part of the backtest's parameters, never of the code:

- At creation: pass `additional_sources` to `create_backtest`, one entry per extra feed:
  `{"symbol": "SPY", "timeframe": "1d"}`, optionally with an `alias` (the short name the strategy uses for
  the feed).
- Later: `update_backtest` with `{"params": {"additional_sources": [...]}}` (send the full list).
- For a multi-timeframe strategy, use the smallest execution/entry timeframe as the backtest's `timeframe`
  and pass every slower context feed through `additional_sources`. Never collapse requested timeframes into
  one feed, and never silently drop a feed the user asked for.
- Run `get_symbol_coverage` for every symbol the backtest will use, not only the primary one.

## Feed naming and access in both engines

The same feed-naming convention (`{SYMBOL}_{TIMEFRAME}`, e.g. `AAPL_1d`, `NVDA_1h`) applies to both engines. Only the access pattern differs:

* **VectorBT (preferred default)** - `data["AAPL_1d"]` returns a `pd.DataFrame` with columns `open/high/low/close/volume` and a datetime index; combine series with normal pandas operations.
* **Backtrader (fallback)** - `self.getdatabyname("AAPL_1d")` returns a feed object whose latest close is `feed.close[0]`.

### VectorBT multi-feed example (cross-asset) - start here

```python
import vectorbt as vbt
import pandas as pd

def signals(data, params):
    # Trade AAPL only when NVDA is in an uptrend (50-bar SMA).
    aapl = data["primary"]              # primary feed (e.g. AAPL_1d)
    nvda = data["NVDA_1d"]              # additional source

    nvda_sma = nvda["close"].rolling(50).mean()
    trend_up = nvda["close"] > nvda_sma  # align to aapl.index - both are 1d

    fast = aapl["close"].rolling(params.get("fast", 10)).mean()
    slow = aapl["close"].rolling(params.get("slow", 30)).mean()
    cross_up   = (fast > slow) & (fast.shift() <= slow.shift())
    cross_down = (fast < slow) & (fast.shift() >= slow.shift())

    entries = cross_up & trend_up.reindex(aapl.index).fillna(False)
    exits   = cross_down | ~trend_up.reindex(aapl.index).fillna(False)
    return entries, exits
```

Notes for VectorBT multi-feed:

* `entries` and `exits` are ALWAYS aligned to `data["primary"].index`. When you combine signals from a different-timeframe feed, `.reindex(data["primary"].index).fillna(False)` (or `.ffill()` for state) - this is the vbt equivalent of "only update HTF state when the HTF bar advances" in Backtrader.
* Pandas does the heavy lifting; no per-bar `next()` callback exists in vbt.
* For mixed-timeframe (e.g. 1m primary + 1d HTF filter), `.reindex(..., method="ffill")` propagates the daily filter across the minute bars.

### Backtrader multi-feed patterns (fallback - only when VectorBT can't express the logic)

The patterns below apply to Backtrader's event-driven `Strategy` class. Use them only when the multi-feed strategy requires path-dependent order management that VectorBT can't handle - otherwise start from the VectorBT example above.

#### Feed naming convention (IMPORTANT)

**All feeds** (both primary and additional) are named using the pattern: `{SYMBOL}_{TIMEFRAME}`

Examples:
- `AAPL_1d` - Apple daily bars (primary)
- `NVDA_1h` - NVIDIA hourly bars (additional)
- `SPY_5m` - SPY 5-minute bars

Supported timeframes:
- Seconds: `1s`, `5s`, `10s`, `30s`
- Minutes: `1m`, `2m`, `3m`, `5m`, `10m`, `15m`, `30m`
- Hours: `1h`, `2h`, `4h`, `6h`, `8h`, `12h`
- Days/Weeks: `1d`, `1w`

Access feeds by name:
```python
aapl_daily = self.getdatabyname("AAPL_1d")
nvda_hourly = self.getdatabyname("NVDA_1h")
```

---

#### Core rules (must follow)

1. **Never assume `self.data` / `self.data0` is "primary" in multi-source jobs.**
   Multi-timeframe engines often add feeds with **smallest timeframe first**, so indexing can change. Always target feeds by **name**.

2. **Always access specific feeds by name:**
   - `self.getdatanames()` → list feed names
   - `self.getdatabyname(name)` → fetch a feed
   - `d._name` → feed name

3. **Orders and positions are per-feed.**
   - Place orders with a feed: `self.buy(data=d, size=1)`, `self.sell(data=d, size=1)`, `self.close(data=d)`
   - Query positions with a feed: `self.getposition(d)`
   - Avoid relying on `self.position` except in single-feed strategies.

4. **Track pending orders per feed** to prevent duplicate orders:
   - `self.orders = {d: None for d in self.datas}`
   - In `notify_order`, clear `self.orders[order.data]` on completion/cancel/reject.

5. **Multi-timeframe logic:** higher timeframe feeds don't advance every `next()`.
   - Detect new HTF bars via `len(htf)` changes (or datetime change).
   - Compute HTF filters/trends when HTF advances, execute trades on the LTF.

6. **Keep strategies generic** (no hardcoded tickers/timeframes inside `Strategy`).
   - Use Strategy params like `primary_name`, `signal_name`, `ltf_name`, `htf_name`.

---

#### Short examples (copyable patterns)

##### A) Build a name→feed map and choose a safe "primary"

```python
import backtrader as bt

class Strategy(bt.Strategy):
    params = dict(primary_name="AAPL_1d")

    def __init__(self):
        self.feeds = {d._name: d for d in self.datas}
        self.names = list(self.feeds.keys())

        # Safe primary selection (name-first)
        self.primary = self.getdatabyname(self.p.primary_name) if self.p.primary_name else self.datas[0]

        # Per-feed pending order tracking
        self.orders = {d: None for d in self.datas}
```

##### B) Multi-symbol: trade every feed independently (buy/sell/close per symbol)

```python
import backtrader as bt

class Strategy(bt.Strategy):
    params = dict(fast=10, slow=30, size=1)

    def __init__(self):
        self.orders = {d: None for d in self.datas}
        self.xovers = {}

        for d in self.datas:
            fast = bt.ind.EMA(d.close, period=self.p.fast)
            slow = bt.ind.EMA(d.close, period=self.p.slow)
            self.xovers[d] = bt.ind.CrossOver(fast, slow)

    def notify_order(self, order):
        if order.status in [order.Completed, order.Canceled, order.Margin, order.Rejected]:
            self.orders[order.data] = None

    def next(self):
        for d in self.datas:
            if self.orders[d]:
                continue

            pos = self.getposition(d)
            sig = self.xovers[d][0]

            if not pos and sig > 0:
                self.orders[d] = self.buy(data=d, size=self.p.size)
            elif pos and sig < 0:
                self.orders[d] = self.close(data=d)
```

##### C) Cross-asset: analyze one feed, trade another

```python
import backtrader as bt

class Strategy(bt.Strategy):
    params = dict(trade_name="AAPL_1d", signal_name="NVDA_1d", size=1)

    def __init__(self):
        self.trade_d = self.getdatabyname(self.p.trade_name)   # traded feed
        self.signal_d = self.getdatabyname(self.p.signal_name) # signal feed

        self.orders = {self.trade_d: None, self.signal_d: None}

        self.signal_sma = bt.ind.SMA(self.signal_d.close, period=50)
        self.fast = bt.ind.EMA(self.trade_d.close, period=10)
        self.slow = bt.ind.EMA(self.trade_d.close, period=30)
        self.x = bt.ind.CrossOver(self.fast, self.slow)

    def notify_order(self, order):
        if order.status in [order.Completed, order.Canceled, order.Margin, order.Rejected]:
            self.orders[order.data] = None

    def next(self):
        if self.orders[self.trade_d]:
            return

        trend_up = self.signal_d.close[0] > self.signal_sma[0]
        pos = self.getposition(self.trade_d)

        if not pos and trend_up and self.x[0] > 0:
            self.orders[self.trade_d] = self.buy(data=self.trade_d, size=self.p.size)
        elif pos and (not trend_up or self.x[0] < 0):
            self.orders[self.trade_d] = self.close(data=self.trade_d)
```

##### D) Multi-timeframe: HTF filter + LTF execution

```python
import backtrader as bt

class Strategy(bt.Strategy):
    params = dict(ltf_name="AAPL_1m", htf_name="AAPL_1d", size=1)

    def __init__(self):
        self.ltf = self.getdatabyname(self.p.ltf_name)
        self.htf = self.getdatabyname(self.p.htf_name)

        self.htf_sma = bt.ind.SMA(self.htf.close, period=50)
        self.ltf_fast = bt.ind.EMA(self.ltf.close, period=10)
        self.ltf_slow = bt.ind.EMA(self.ltf.close, period=30)
        self.ltf_x = bt.ind.CrossOver(self.ltf_fast, self.ltf_slow)

        self.orders = {self.ltf: None, self.htf: None}
        self._last_htf_len = len(self.htf)
        self.htf_trend_up = False

    def notify_order(self, order):
        if order.status in [order.Completed, order.Canceled, order.Margin, order.Rejected]:
            self.orders[order.data] = None

    def next(self):
        # Update HTF state only when HTF advances
        if len(self.htf) != self._last_htf_len:
            self._last_htf_len = len(self.htf)
            self.htf_trend_up = self.htf.close[0] > self.htf_sma[0]

        if self.orders[self.ltf]:
            return

        pos = self.getposition(self.ltf)

        if not pos and self.htf_trend_up and self.ltf_x[0] > 0:
            self.orders[self.ltf] = self.buy(data=self.ltf, size=self.p.size)
        elif pos and (not self.htf_trend_up or self.ltf_x[0] < 0):
            self.orders[self.ltf] = self.close(data=self.ltf)
```

---

#### Backtrader-specific agent generation rules (when translating user intent into strategy code)

- If multi-source is implied or explicit: default to **name-based** feed access (`getdatabyname`) and **per-feed** order/position tracking.
- If user says "primary/secondary" without names: implement Strategy params (`primary_name`, `secondary_name`) with safe fallbacks.
- Always place orders with explicit `data=...` when multiple feeds exist.
- For MTF: compute HTF signals only on new HTF bars (`len(htf)` changes) and trade on LTF.
