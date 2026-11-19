# ML training runs

Train machine learning models on historical market data and use them in backtest strategies. This guide covers
the complete workflow: create a training run, write training code, run training, check results, and use the
trained model in a backtest.

When the user says anything ML related (for example "create a random forest strategy"), read this whole file
first. An ML "strategy" request is two jobs: the training run AND a backtest that uses the trained model. Do
both in the same conversation without waiting to be asked again, unless training failed or the user wants the
training only.

## Contents

- Tools
- Statuses
- Complete workflow (steps 1 to 6)
- Training code contract
- Available helpers (`bb_training`, `bb_models`)
- Complete training code template
- Complete backtest strategy template (using a trained model), feature consistency and normalization rules
- Data splitting
- File management
- Error handling and debugging, common errors
- Scenario playbooks
- Saving models
- Key rules

---

## Tools

| Action | Tool |
|--------|------|
| Create training run | `create_training_run` |
| List training runs | `list_training_runs` |
| Get training run (status, metrics, log tail) | `get_training_run` |
| Update training run | `update_training_run` |
| Delete training run | `delete_training_run` |
| Run training | `run_training` |
| Stop training | `stop_training` |
| Test trained model | `test_trained_model` |
| Get results | `get_training_results` |
| List, read, write, edit, delete, rename files, create a directory | `list_files`, `read_file`, `write_file`, `edit_file`, `multiedit_file`, `delete_file`, `rename_file`, `create_directory`, each with `target_type: "training"` and `target_id: <training run id>` |

`glob_files`, `grep_files` and `check_python_syntax` work on backtests only, not on training runs.

---

## Statuses

| Status | Meaning |
|--------|---------|
| `draft` | Created, not yet run |
| `queued` | Queued for execution |
| `running` | Currently training |
| `completed` | Training finished successfully |
| `failed` | Training encountered an error |
| `stopped` | User stopped training |
| `testing` | Running test evaluation on held-out data |

---

## Complete workflow

### Step 1: Create a training run

Check the data first: `get_symbol_coverage` for the symbol, so the window fits the data and the user's plan.

```
create_training_run
{
  "name": "SPY Direction Classifier",
  "description": "Random Forest predicting next-day price direction using SMA, EMA, RSI",
  "symbol": "SPY",
  "timeframe": "1d",
  "start_date": "2020-01-01",
  "end_date": "2024-12-31",
  "test_split": 0.2,
  "val_split": 0.1,
  "requires_gpu": false
}
```

**Response** includes the `id` (e.g. `42`). Store and reuse this ID for all subsequent calls.

**Parameter reference:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | str | Yes | Display name |
| `description` | str | No | Short description |
| `symbol` | str | Yes | Trading symbol (SPY, AAPL, TSLA, etc.) |
| `timeframe` | str | Yes | Data timeframe (1d, 1h, 5m, 15m, 30m, 4h, etc.) |
| `start_date` | str | Yes | Start date YYYY-MM-DD |
| `end_date` | str | Yes | End date YYYY-MM-DD |
| `test_split` | float | No | Fraction held out for test (0-0.5, default 0.2) |
| `val_split` | float | No | Fraction of train data for validation (0-0.3, default 0.0) |
| `requires_gpu` | bool | No | Whether a GPU is needed (default: false) |
| `required_gpu_type` | str | No | GPU type when `requires_gpu` is true: any, t4, l4, a10g, a100, h100 |
| `code` | str | No | Content of `train.py`; written as `train.py` right after creation |

**GPU guidance:**
- `requires_gpu: false` - sklearn, xgboost, lightgbm (no GPU needed, faster queue). This is the default and the common case.
- `requires_gpu: true` with `t4` or `any` - small PyTorch/TensorFlow models
- `requires_gpu: true` with `a10g`/`l4` - medium models
- `requires_gpu: true` with `a100`/`h100` - large models
- GPU access and the available GPU types depend on the user's plan (`get_account_status`). When the plan has no
  GPU, do not attempt the GPU run and report a failure: run the same work on CPU (simple models such as logistic
  regression or small forests train in seconds on CPU) and say in one line that you did.

### Step 2: Write the training code

Either pass the code as `code` in `create_training_run`, or write `train.py` to the training run's files:

```
write_file
{
  "target_type": "training",
  "target_id": 42,
  "path": "train.py",
  "content": "<full Python code - see template below>"
}
```

**Do not write a `requirements.txt` for the standard environment.** These packages are pre-installed and need
no file: `numpy`, `pandas`, `polars`, `pyarrow`, `scipy`, `numba`, `statsmodels`, `scikit-learn`, `lightgbm`,
`joblib`, `matplotlib`, `seaborn`, `backtrader`, `vectorbt`. A `requirements.txt` is only for a package outside
that set which the user explicitly asked for (`torch`, `tensorflow`, `xgboost`, `catboost`):

```
write_file
{
  "target_type": "training",
  "target_id": 42,
  "path": "requirements.txt",
  "content": "xgboost"
}
```

Where the server has extra packages switched off, a run that carries a `requirements.txt` fails with "Custom
dependency installation is disabled on this server". Then delete the file (`delete_file`), tell the user in one
line that the package is not available, and train with the closest pre-installed model (`lightgbm` or
`scikit-learn`).

### Step 3: Run training

```
run_training
{ "training_run_id": 42 }
```

Training is asynchronous. Poll the run every 5-10 seconds:

```
get_training_run
{ "training_run_id": 42 }
```

Check the `status` field until it reaches `completed`, `failed`, or `stopped`. Training uses credits.

### Step 4: Check results

```
get_training_results
{ "training_run_id": 42 }
```

Returns:
```json
{
  "id": 42,
  "status": "completed",
  "metrics": {
    "accuracy": 0.62,
    "val_accuracy": 0.58,
    "test_accuracy": 0.55,
    "train_samples": 700,
    "val_samples": 100,
    "test_samples": 200,
    "execution_time_seconds": 15.3
  },
  "model_artifact_path": "/io/artifacts/1/training_42/",
  "execution_time_seconds": 15.3
}
```

For full details, including the status message and the tail of the execution output:
```
get_training_run
{ "training_run_id": 42 }
```

### Step 5: Run test evaluation (optional)

If the training run completed and has a saved model, run the `test()` function separately on held-out test data:

```
test_trained_model
{ "training_run_id": 42 }
```

Requires `status == "completed"` and a saved model. The test is asynchronous too: the run shows status `testing`
until it finishes (poll `get_training_run`). The values `test()` returned then appear in `get_training_results`
with a `test_` prefix, and `test_error` says why if it failed.

### Step 6: Use the trained model in a backtest strategy

Create a normal backtest (`create_backtest`) with a strategy that loads the trained model, and pass the training
run's id as `training_run_id`. See the **backtest strategy template** below.

**CRITICAL:** The features fed to the model at inference (in the strategy) MUST be the SAME indicators computed in
the SAME order as during training. This is the most common source of bugs.

---

## Training Code Contract

The file `train.py` must define:

```python
def train(df: pd.DataFrame, artifact_dir: str, params: dict) -> dict:
```

**Arguments:**

| Argument | Type | Description |
|----------|------|-------------|
| `df` | pd.DataFrame | Training data. **Index:** tz-naive DatetimeIndex named `'datetime'`. **Columns:** `['timestamp', 'open', 'high', 'low', 'close', 'volume', 'symbol', 'instrument_id']`. Sorted chronologically. Pass directly to `compute_indicators(df, ...)` - do NOT add a `datetime` column (it's already the index). |
| `artifact_dir` | str | Directory path for saving models. Persists across runs. Accessible via `load_trained_model()` in strategies. |
| `params` | dict | Training parameters. Contains `symbol`, `timeframe`, `start_date`, `end_date`, `test_split`, `val_split`. If `val_split > 0`, also contains `params['val_data']` as a validation DataFrame (same format as `df`). |

**Returns:** Dict of metrics. Example: `{"accuracy": 0.62, "loss": 0.45}`. The platform automatically adds metadata like `train_samples`, `execution_time_seconds`.

**Optional test function (same file):**

```python
def test(df: pd.DataFrame, artifact_dir: str, params: dict) -> dict:
```

Same signature but receives the held-out test data. Return keys are auto-prefixed with `test_` if not already present (e.g. returning `{"accuracy": 0.55}` becomes `{"test_accuracy": 0.55}` in metrics).

**No side effects at import time.** The platform imports `train.py` before calling `train()`. The module must
import cleanly: no `os.makedirs`, file writes, downloads or network calls at module level. Do that work inside
`train()` and write every artifact into `artifact_dir` only. (A real failure: a module-level `os.makedirs("output")`
ended the run with "Training entry point failed to load ... PermissionError".)

---

## Available Helpers

Two helper modules are auto-injected. No install needed - just import.

### bb_training - Indicator Computation

```python
from bb_training import compute_indicators

enriched_df = compute_indicators(df, {
    'sma_20': {'type': 'SMA', 'period': 20},
    'ema_10': {'type': 'EMA', 'period': 10},
    'rsi_14': {'type': 'RSI', 'period': 14},
    'bb_upper': {'type': 'BBands', 'period': 20, 'output': 'top'},
    'bb_lower': {'type': 'BBands', 'period': 20, 'output': 'bot'},
    'atr_14': {'type': 'ATR', 'period': 14},
    'macd':    {'type': 'MACD'},
    'adx_14':  {'type': 'ADX', 'period': 14},
    'cci_14':  {'type': 'CCI', 'period': 14},
})
```

Uses a real backtrader cerebro internally - indicator values are **identical** to those in a `bt.Strategy`. Supports all `bt.indicators` types (SMA, EMA, RSI, MACD, BBands, ATR, Stochastic, CCI, ROC, OBV, PSAR, ADX, etc.). Use the `'output'` key to select a named line (e.g. `'top'` for BBands upper band, `'bot'` for lower).

**Important rules for `compute_indicators`:**
- Pass `df` directly as received in `train()` - do NOT add a `'datetime'` column (the datetime is already the index).
- The function needs enough rows for indicator warmup. An indicator with `period: 200` needs at least 200 rows before producing values. If the input DataFrame is too small, the function will raise a `ValueError`.
- This is especially important for `val_data` - with `val_split: 0.1` and 1000 total bars, `val_data` is only ~100 rows. An EMA with period 200 will fail on val_data. **Keep indicator periods shorter than your smallest data split**, or wrap val_data processing in a try/except.

### bb_models - Model Loading

```python
from bb_models import load_trained_model

model = load_trained_model(training_run_id=42, filename="model.pkl")
```

Available in **both** training runs **and** backtest strategies. Loads pickle files from `/io/artifacts/{user_id}/training_{training_run_id}/{filename}`.

Default filename is `"model.pkl"`. Pass a different filename if you saved with a different name (e.g. `filename="scaler.pkl"`).

To use a trained model in a backtest: load it ONCE (Backtrader: in `__init__`; VectorBT: at module level, outside
`signals()`), and ALSO set the backtest's `training_run_id` parameter to the same id. The parameter is what makes
the platform stage the artifacts for the run and route it to a GPU when needed; without it the model file is not
there.

---

## Complete Training Code Template

Use this as a starting point. Adapt indicators, model type, and target to the user's needs.

```python
# train.py
import pandas as pd
import numpy as np
import pickle
from bb_training import compute_indicators


# ── CONFIGURATION ──────────────────────────────────────────────────────────
# Raw indicators - computed by compute_indicators().
# The SAME indicators (type + period) must be replicated in the strategy.
INDICATORS = {
    'sma_20': {'type': 'SMA', 'period': 20},
    'ema_10': {'type': 'EMA', 'period': 10},
    'rsi_14': {'type': 'RSI', 'period': 14},
}

# IMPORTANT: Use NORMALIZED features, not raw indicator values.
# Raw price-level indicators (SMA, EMA, BBands, etc.) don't generalize
# across different price ranges. Convert them to ratios relative to close.
# RSI, ROC, and other bounded/normalized indicators can be used directly.
FEATURE_COLS = [
    'sma_20_ratio',   # close / sma_20 - 1  (distance from SMA as fraction)
    'ema_10_ratio',   # close / ema_10 - 1
    'rsi_14',         # already normalized (0-100), use directly
]


# ── HELPERS ────────────────────────────────────────────────────────────────

def _prepare_features(df):
    """Compute indicators, normalize price-level features, build X and y."""
    enriched = compute_indicators(df, INDICATORS)

    # Normalize price-level indicators as ratios relative to close
    enriched['sma_20_ratio'] = enriched['close'] / enriched['sma_20'] - 1
    enriched['ema_10_ratio'] = enriched['close'] / enriched['ema_10'] - 1
    # RSI is already bounded (0-100) - no normalization needed

    # Target: 1 if next bar closes higher, 0 otherwise
    enriched['_target'] = (enriched['close'].shift(-1) > enriched['close']).astype(int)
    enriched = enriched.dropna()

    X = enriched[FEATURE_COLS].values
    y = enriched['_target'].values
    return X, y


# ── REQUIRED: train() ─────────────────────────────────────────────────────

def train(df, artifact_dir, params):
    from sklearn.ensemble import RandomForestClassifier
    from sklearn.metrics import accuracy_score

    X_train, y_train = _prepare_features(df)

    model = RandomForestClassifier(n_estimators=100, random_state=42)
    model.fit(X_train, y_train)

    train_acc = accuracy_score(y_train, model.predict(X_train))

    # Save model to artifact_dir (REQUIRED for load_trained_model to work)
    with open(f"{artifact_dir}/model.pkl", "wb") as f:
        pickle.dump(model, f)

    metrics = {
        "accuracy": round(train_acc, 4),
        "features": FEATURE_COLS,
        "n_samples": len(X_train),
    }

    # Handle validation data if provided (val_split > 0)
    # Wrap in try/except: val_data may be too small for indicator warmup
    val_data = params.get('val_data')
    if val_data is not None and len(val_data) > 0:
        try:
            X_val, y_val = _prepare_features(val_data)
            if len(X_val) > 0:
                val_acc = accuracy_score(y_val, model.predict(X_val))
                metrics["val_accuracy"] = round(val_acc, 4)
        except ValueError as e:
            print(f"Skipping validation: {e}")

    return metrics


# ── OPTIONAL: test() ───────────────────────────────────────────────────────

def test(df, artifact_dir, params):
    """Evaluate on held-out test data. Called automatically if test_split > 0."""
    from sklearn.metrics import accuracy_score

    X_test, y_test = _prepare_features(df)

    # Guard against empty data after indicator warmup + dropna
    if len(X_test) == 0:
        return {"accuracy": None, "samples": 0, "note": "Insufficient data after indicator warmup"}

    with open(f"{artifact_dir}/model.pkl", "rb") as f:
        model = pickle.load(f)

    preds = model.predict(X_test)
    acc = accuracy_score(y_test, preds)

    return {
        "accuracy": round(acc, 4),
        "samples": len(X_test),
    }
```

**requirements.txt:** none. `scikit-learn` is pre-installed.

---

## Complete Backtest Strategy Template (Using Trained Model)

After training run #42 completes, create a normal backtest with this strategy code: `create_backtest` with the code and `training_run_id: 42`, then `check_python_syntax` and `run_backtest`.

```python
# strategy.py
import backtrader as bt
import numpy as np
from bb_models import load_trained_model

TRAINING_RUN_ID = 42  # ← Replace with actual training run ID


class Strategy(bt.Strategy):
    params = dict(
        stake=100,
    )

    def __init__(self):
        # Load the trained model from the completed training run
        self.model = load_trained_model(TRAINING_RUN_ID)

        # CRITICAL: Must match training indicators EXACTLY
        # Same types, same periods, same order as INDICATORS dict in train.py
        self.sma_20 = bt.ind.SMA(self.data.close, period=20)
        self.ema_10 = bt.ind.EMA(self.data.close, period=10)
        self.rsi_14 = bt.ind.RSI(self.data.close, period=14)

        self.order = None

    def notify_order(self, order):
        if order.status in [order.Submitted, order.Accepted]:
            return
        if order.status in [order.Completed, order.Canceled, order.Margin, order.Rejected]:
            self.order = None

    def next(self):
        if self.order:
            return

        # Build NORMALIZED feature vector - SAME order as training FEATURE_COLS
        # Price-level indicators are converted to ratios (close / indicator - 1)
        close = self.data.close[0]
        features = np.array([[
            close / self.sma_20[0] - 1,   # sma_20_ratio
            close / self.ema_10[0] - 1,   # ema_10_ratio
            self.rsi_14[0],               # already normalized
        ]])

        prediction = self.model.predict(features)[0]

        if prediction == 1 and not self.position:
            self.order = self.buy(size=self.p.stake)
        elif prediction == 0 and self.position:
            self.order = self.close()
```

### CRITICAL: Feature Consistency Rule

The indicator definitions in the strategy **must exactly match** what `compute_indicators` produced during training, **including the normalization step**:

| Training (`_prepare_features`) | Strategy (`next()`) |
|--------------------------------|---------------------|
| `enriched['close'] / enriched['sma_20'] - 1` | `close / self.sma_20[0] - 1` |
| `enriched['close'] / enriched['ema_10'] - 1` | `close / self.ema_10[0] - 1` |
| `enriched['rsi_14']` (direct) | `self.rsi_14[0]` (direct) |

The order in the `features` array must match the order in `FEATURE_COLS` from training. If training used `[sma_20_ratio, ema_10_ratio, rsi_14]`, the strategy must compute and pass them in that exact order.

**If the features don't match (wrong order, missing normalization, or wrong indicators), the model will produce garbage predictions with no error message.** This is the #1 source of bugs.

### CRITICAL: Feature Normalization Rule

**Always normalize price-level indicators before feeding them to the model.** Raw indicator values (SMA, EMA, BBands, VWAP, etc.) are on the same scale as the asset price. A model trained on AAPL at $150 will fail on AAPL at $200 - or produce heavily biased probability outputs (e.g. always <30% or always >70%).

**Normalize by converting to ratios relative to close:**
```python
# In _prepare_features():
enriched['sma_20_ratio'] = enriched['close'] / enriched['sma_20'] - 1
enriched['ema_10_ratio'] = enriched['close'] / enriched['ema_10'] - 1
# In strategy next():
close / self.sma_20[0] - 1
```

**Indicators that are already normalized (use directly, no conversion needed):**
- RSI (0-100), ROC (percentage), ATR ratio (ATR/close), Stochastic (0-100), Williams %R (-100-0), ADX (0-100), CCI (unbounded but mean-reverting around 0)

**Indicators that MUST be normalized:**
- SMA, EMA, WMA, DEMA, TEMA, KAMA (all moving averages)
- BBands upper/middle/lower → use `(close - bb_mid) / (bb_upper - bb_lower)` or `close / bb_mid - 1`
- VWAP → use `close / vwap - 1`
- Any custom indicator on the price scale

**Why ratio-based normalization works across symbols and date ranges:**
The `close / indicator - 1` ratio is computed per bar using only that bar's own values - no fitted scaler, no training-set statistics. "Price is 3% above its SMA-20" means the same thing whether the stock is at $50 or $500, or whether you're backtesting 2022 or 2025. This is why ratio-based normalization generalizes and fitted scalers do not.

**Do NOT use `StandardScaler`, `MinMaxScaler`, or any fitted scaler.** These learn mean/std or min/max from training data and must be saved as a separate artifact, then loaded at inference. If the strategy runs on a different price range, the scaling breaks. Ratio-based normalization avoids this entirely.

---

## Data Splitting

Data is split **chronologically** (not randomly - time series data must preserve order).

Given a dataset of N bars with `test_split=0.2` and `val_split=0.1`:

```
|←--- Training (72%) ---→|←- Val (8%) -→|←-- Test (20%) --→|
|    bars 0-719           | bars 720-799  |  bars 800-999    |
```

1. **Test set**: Last `N × test_split` bars are held out (bars 800-999)
2. **From remaining**: Last `remaining × val_split` bars become validation (bars 720-799)
3. **Rest**: Training data (bars 0-719)

- `train()` receives the training portion as `df`
- If `val_split > 0`, validation data is in `params['val_data']`
- `test()` receives the test portion as `df`

---

## File management

Training runs use dedicated file storage. The primary files:

| File | Purpose |
|------|---------|
| `train.py` | Main training script (must define `train()`) |
| `requirements.txt` | Only for packages outside the standard environment, one per line (see Step 2) |
| Any `.py` files | Additional modules importable from `train.py` |

All file tools take `target_type: "training"` and `target_id: <training run id>`.

### Write a file
```
write_file
{ "target_type": "training", "target_id": 42, "path": "train.py", "content": "..." }
```

### Read a file
```
read_file
{ "target_type": "training", "target_id": 42, "path": "train.py" }
```

### Edit a file (single replacement)
```
edit_file
{
  "target_type": "training",
  "target_id": 42,
  "path": "train.py",
  "old_string": "n_estimators=100",
  "new_string": "n_estimators=200"
}
```

### Multi-edit a file (atomic batch)
```
multiedit_file
{
  "target_type": "training",
  "target_id": 42,
  "path": "train.py",
  "edits": [
    {"old_string": "n_estimators=100", "new_string": "n_estimators=200"},
    {"old_string": "random_state=42", "new_string": "random_state=0"}
  ]
}
```

### List files
```
list_files
{ "target_type": "training", "target_id": 42 }
```

### Delete a file
```
delete_file
{ "target_type": "training", "target_id": 42, "path": "train.py" }
```

---

## Error handling and debugging

### When training fails:

1. **Get the training run** (`get_training_run`) and read its status message, the tail of the execution output
   and the metrics.

2. **Read the current code:** `read_file` with `path: "train.py"`.

3. **Diagnose** the error from the traceback.

4. **Fix the code** using `edit_file` (small changes) or `write_file` (rewrite).

5. **Re-run** with `run_training` and poll `get_training_run` again.

### Common errors

| Error | Cause | Fix |
|-------|-------|-----|
| `No train() function found` | Missing or misspelled `train` function | Ensure `def train(df, artifact_dir, params):` exists in `train.py` |
| `No training code provided` | `train.py` is empty or missing | Write `train.py` to file storage |
| `No market data returned` | Invalid symbol, date range, or timeframe | Check parameters match supported values |
| `ModuleNotFoundError: xxx` | The package is not in the standard environment | Use a pre-installed equivalent, or add the package to `requirements.txt` when the user asked for it |
| `Custom dependency installation is disabled on this server` | The run carries a `requirements.txt` and the server has extra packages switched off | Delete `requirements.txt`, tell the user the package is not available, use a pre-installed equivalent |
| `Training run has no parameters` | `params` not set on the training run | Update the training run with valid params |
| `Training run is already running` | Duplicate run attempt | Wait for current run to finish or stop it first |
| `cannot insert datetime, already exists` | Code added a `datetime` column before calling `compute_indicators` | Do NOT add a `datetime` column - the DataFrame already has a DatetimeIndex. Pass `df` directly to `compute_indicators()` |
| `compute_indicators produced no rows` | Indicator warmup period exceeds available data (e.g. EMA 200 on 100 rows) | Use shorter indicator periods or more data. Wrap val_data in `try/except ValueError` |
| `test_error: 0 samples after prepare` | Test split too small for indicator warmup (e.g. EMA-50 on 201 rows) | Add `if len(X_test) == 0: return {...}` guard in `test()`, or use shorter indicator periods, or increase data range |
| Model always predicts same class / probability stuck near 0 or 1 | Raw price-level features (SMA, EMA) not normalized | Convert to ratios: `close / indicator - 1`. See Feature Normalization Rule |
| Feature shape mismatch (in strategy) | Strategy features don't match training | Ensure same indicators, same periods, same order, same normalization |
| `Model not found` (in strategy) | Training incomplete or wrong ID | Verify training run completed and has `model_artifact_path` |
| `Training entry point failed to load` with a `PermissionError` or similar | Code runs at import time (creates folders, writes files, calls the network) | Move that work inside `train()`; write only into `artifact_dir` |

---

## Scenario playbooks

### A) User asks "Train an ML model to predict [SYMBOL] direction"

1. **Create** the training run via `create_training_run`:
   - Symbol as requested, timeframe: `1d` unless specified
   - Date range: 2-3 years of data (e.g. 2022-01-01 to 2024-12-31), inside what `get_symbol_coverage` allows
   - `test_split: 0.2`, `val_split: 0.1`
   - `requires_gpu: false` for sklearn models

2. **Write `train.py`** (as `code` at creation, or with `write_file`) using the template above. Adapt indicators to
   the user's request.

3. **No `requirements.txt`**: `scikit-learn` and `lightgbm` are pre-installed.

4. **Run** via `run_training`, poll `get_training_run`, and tell the user how it is going.

5. **Report results**: accuracy, val_accuracy, test_accuracy, execution time. Report them as they are, including
   weak validation or test numbers.

6. **Continue** to a backtest that uses the trained model (playbook B) when the user asked for a strategy; offer
   it when they asked for the training only.

### B) User asks "Use my trained model in a backtest"

1. **Get** the training run via `get_training_run` to verify it completed:
   - Check `status == "completed"` and `model_artifact_path` is set.

2. **Read `train.py`** via `read_file` to identify which indicators and features were used.

3. **Create a normal backtest** with a strategy that:
   - Imports and calls `load_trained_model(TRAINING_RUN_ID)` in `__init__`
   - Computes the SAME indicators using `bt.ind.*`
   - Passes features to `model.predict()` in the SAME order as training
   - **Pass `training_run_id` to `create_backtest`** so the backtest is linked to the training run (see key rule 14)

4. **Set the backtest** to a **different date range** than training for out-of-sample evaluation.

5. **Run the backtest** (`check_python_syntax`, then `run_backtest`) and report results.

### C) Training failed - user asks to fix it

1. **Get** the training run via `get_training_run` - examine the status message and the output tail.

2. **Read** `train.py` via `read_file` to see the current code.

3. **Diagnose** the error from the traceback.

4. **Fix** using `edit_file` (small fix) or `write_file` (rewrite).

5. **Re-run** via `run_training`. Repeat if needed (max 3 attempts).

### D) User asks to improve model accuracy

1. **Get results** via `get_training_results` to see current metrics.

2. **Read** `train.py` via `read_file` to understand the current approach.

3. **Suggest and implement** improvements:
   - Add more indicators (ATR, MACD, BBands, ADX, etc.)
   - Try a different model (LightGBM, GradientBoosting, RandomForest, all pre-installed)
   - Tune hyperparameters (n_estimators, max_depth, learning_rate)
   - Add feature engineering (lag features, rolling stats, price ratios)
   - Increase training data date range

4. **Update** `train.py` via `edit_file` or `write_file`. Stay inside the pre-installed packages (Step 2).

5. **Re-run** via `run_training` and compare metrics to the previous run.

### E) User asks for deep learning / PyTorch training

1. **Create** via `create_training_run` with `requires_gpu: true`, `required_gpu_type: "t4"` or higher (when the
   user's plan includes that GPU).

2. **Write `requirements.txt`** via `write_file` with `torch` (or `tensorflow`). Neither is pre-installed, so
   this is the one case that needs the file. If the run fails with "Custom dependency installation is disabled
   on this server", deep-learning packages are not available there: delete the file, say so, and offer a
   pre-installed model instead (`scikit-learn`'s `MLPClassifier` or `lightgbm`, on CPU).

3. **Write `train.py`** via `write_file` using PyTorch. Save model with:
   ```python
   torch.save(model.state_dict(), f"{artifact_dir}/model.pt")
   ```

4. **Note for the strategy**: For PyTorch models, `load_trained_model()` won't work since it uses `pickle.load()`.
   Instead, the strategy must load directly:
   ```python
   import torch, os
   model_path = f"/io/artifacts/{os.environ['_ARTIFACT_USER_ID']}/training_{TRAINING_RUN_ID}/model.pt"
   model = MyModel()
   model.load_state_dict(torch.load(model_path))
   model.eval()
   ```

### F) User asks about, or wants to change, the training behind an existing backtest

1. Read the backtest (`get_backtest`) and take `params.training_run_id`.

2. Read the training run (`get_training_run`) and its `train.py` (`read_file`) to understand the current feature
   set before changing anything.

3. Change, re-run and re-test as in playbooks C and D, then re-run the backtest so it uses the new model.

---

## Saving Models

In the `train()` function, save to `artifact_dir`:

```python
# sklearn / pickle-compatible models (recommended)
import pickle
with open(f"{artifact_dir}/model.pkl", "wb") as f:
    pickle.dump(model, f)

# Save additional artifacts if needed
with open(f"{artifact_dir}/scaler.pkl", "wb") as f:
    pickle.dump(scaler, f)

# PyTorch
import torch
torch.save(model.state_dict(), f"{artifact_dir}/model.pt")
```

Default filename for `load_trained_model()` is `"model.pkl"`. For other filenames: `load_trained_model(42, filename="scaler.pkl")`.

---

## Key rules

1. **Feature consistency is non-negotiable.** Training and inference must use identical indicators in identical order, **including normalization**. Document the feature list in the training code clearly (use a `FEATURE_COLS` constant).

2. **Always normalize price-level features.** Never feed raw SMA/EMA/BBands values to a model - convert to ratios (`close / indicator - 1`). Raw values don't generalize across price ranges and produce biased predictions. RSI, ROC, Stochastic, and similar bounded indicators are already normalized.

3. **`train.py` must define `def train(df, artifact_dir, params) -> dict`.** This is the only hard requirement for the code, together with importing without side effects.

4. **Do NOT modify `df` before passing to `compute_indicators()`.** The DataFrame already has a DatetimeIndex named `'datetime'`. Do not add a `datetime` column - it will cause errors. Pass `df` directly.

5. **Indicator periods must fit your smallest data split.** With `val_split: 0.1` and 1000 bars, val_data is only ~100 rows. An indicator with `period: 200` will fail on it. Wrap val_data processing in `try/except ValueError`, or keep periods shorter than the smallest split. The same applies to `test()`.

6. **The model must be saved to `artifact_dir`** for it to persist. Use `pickle.dump()` for sklearn-compatible models.

7. **Chronological splits only.** Never shuffle time series data. The platform handles splitting automatically via `test_split` and `val_split`.

8. **Pre-installed packages:** `numpy`, `pandas`, `polars`, `pyarrow`, `scipy`, `numba`, `statsmodels`, `scikit-learn`, `lightgbm`, `joblib`, `matplotlib`, `seaborn`, `backtrader`, `vectorbt`. Do not write a `requirements.txt` for these. Anything else (`torch`, `tensorflow`, `xgboost`, `catboost`) needs one and works only where the server allows extra packages (Step 2).

9. **Use `requires_gpu: false` for sklearn and lightgbm.** Avoids unnecessary GPU costs and queue times.

10. **Training and testing are asynchronous.** After `run_training` or `test_trained_model`, poll `get_training_run` until the status is terminal before reading results.

11. **Read the run's output for debugging.** The training run returned by `get_training_run` carries the status message and the tail of the execution output.

12. **Guard `test()` against empty data.** After indicator warmup + `dropna()`, the test split may have 0 rows. Always check `if len(X_test) == 0` and return graceful metrics instead of crashing.

13. **Check `test_status` and `test_error` in results.** `get_training_results` returns top-level `test_status` ("completed"/"failed"/null) and `test_error` (error message) fields. If `test_error` is set, the test evaluation failed - fix the code and re-run.

14. **Always link backtests to their training run.** When creating a backtest that uses a trained model, pass `training_run_id` to `create_backtest` (it is stored in the backtest's params, e.g. `params.training_run_id = 15`). This stages the model for the run and lets you (and the user) trace back from any backtest to the training run it depends on. When a user asks about the training for an existing backtest, check `params.training_run_id` first.

15. **Never train inside a strategy.** Training happens in a training run; the strategy only loads the saved model.
