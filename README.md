# BacktestLoop for Claude

Backtest trading strategies from a conversation. You describe a strategy in plain words, Claude writes the
strategy code, and [BacktestLoop](https://backtestloop.com) supplies the market data, runs the backtest and
stores the results in your BacktestLoop account.

BacktestLoop only simulates trading on historical or live market data. It cannot place real orders or connect
to a broker, and nothing it reports is personal investment advice.

## What is in the plugin

- **The BacktestLoop connector** (`https://api.backtestloop.com/mcp-code`): the tools Claude uses to create and
  run backtests, read results, run parameter experiments and out-of-sample tests, analyse market data, train
  machine-learning models and manage paper trading.
- **The `backtesting` skill**: the rules for writing strategies that run correctly on BacktestLoop. It covers
  both engines (VectorBT and Backtrader), the platform's strategy contract, the step-by-step workflows, the
  indicator registries, the ML training guide and how results are read and reported. Claude loads it on its own
  when you ask for backtesting work.

Apart from its icon, the plugin holds only text: the skill's instructions and the address of the connector. It
has no hooks, scripts or programs, so it runs nothing on your computer.

## What you need

A BacktestLoop account (free to start) at <https://app.backtestloop.com>. The first time Claude uses the
connector you sign in to BacktestLoop and approve the connection. Backtests, experiments and training use
credits from your BacktestLoop balance; paper trading needs a paid plan.

## Install

In Claude Code, run:

```
/plugin install backtestloop --marketplace VectorSpell/backtestloop-claude-plugin
```

Claude Code versions before 2.1.275 need two steps:

```
/plugin marketplace add VectorSpell/backtestloop-claude-plugin
/plugin install backtestloop@backtestloop
```

In the Claude app on the web and desktop, the guide at <https://backtestloop.com/docs/claude-integration>
shows how to add the plugin and connect BacktestLoop.

## How to use it

Ask in plain words, for example:

- "Backtest a 20/50 EMA crossover on TSLA, 1 hour bars, last 6 months."
- "Add a 3% stop loss and compare stops of 2, 3 and 5%."
- "Test this strategy on other years to see if it holds up."
- "What is the average daily range of NQ futures by weekday this year?"
- "Train a random forest to predict SPY's next-day direction and backtest it."

You can also paste TradingView Pine Script, written trading rules or a YouTube link to a strategy video.

In the Claude app on the web and desktop, results come with charts: the equity curve with its drawdown, and bar
charts for comparisons and out-of-sample years. In Claude Code you get the numbers and a link to the backtest's
charts in BacktestLoop.

The setup steps, more examples and troubleshooting are in the guide at
<https://backtestloop.com/docs/claude-integration>.

## What data is sent

The connector sends BacktestLoop what the work needs: the strategy, analysis and model-training code Claude
writes, the parameters of each backtest (symbol, timeframe, dates, settings), the requests to read your
backtests and results, and for a video question the YouTube link and your question. Nothing else from your
conversation is sent. Everything is stored in your own BacktestLoop account, where you can view and delete it.
See the privacy policy at <https://backtestloop.com/privacy>.

## Support

support@backtestloop.com
