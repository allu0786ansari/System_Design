# Study Notes — Tier 0 & Tier 1

**Project:** Portfolio_Optimization (RL-based portfolio allocation)
**Tier 0:** Python & ML Basics — Foundation
**Tier 1:** Finance Fundamentals — Speak the language of quant interviews

> Each topic below explains the concept deeply, gives a plain-English intuition,
> a worked example, and then shows **exactly where and how it's used** in this
> project's real code so you can defend it in an interview.

---

## TIER 0 — Python & ML Basics

### 1. NumPy — Arrays, Broadcasting, Vectorised Operations

**The core idea**

A NumPy array is a grid of numbers that all share the same type, stored in one
contiguous block of memory. This is different from a Python list, which is a
collection of pointers scattered around memory. Because a NumPy array is one
solid block, the CPU can process it using vectorised instructions (SIMD) —
applying the same operation to many numbers at once instead of looping one
element at a time in Python.

**Why this matters practically**

A Python `for` loop over a list of a million numbers is slow because Python
has to check the type of every single element on every iteration. A NumPy
operation on the same million numbers runs the loop in pre-compiled C code,
which is typically 10x–100x faster.

```python
# Slow: pure Python loop
returns = [0.01, -0.02, 0.015, ...]  # a million elements
squared = [r ** 2 for r in returns]

# Fast: vectorised NumPy
import numpy as np
returns = np.array([0.01, -0.02, 0.015, ...])
squared = returns ** 2   # no explicit loop — happens in C
```

**Broadcasting**

Broadcasting is NumPy's rule for applying an operation between arrays of
different shapes, by "stretching" the smaller one without actually copying
data. The classic example: subtracting a single mean value from every row of
a matrix.

```python
prices = np.array([[100, 200], [102, 198], [101, 205]])  # shape (3, 2)
mean_price = prices.mean(axis=0)                          # shape (2,)
centered = prices - mean_price   # broadcasting: (3,2) - (2,) works automatically
```

The rule: NumPy compares shapes from the right. Dimensions are compatible if
they're equal, or if one of them is 1. Here `(3, 2)` and `(2,)` broadcast
because the trailing dimension `2` matches.

**Real-world example**

Imagine you have daily returns for 40 stocks over 500 days — a `(500, 40)`
matrix. You want the portfolio return for each day, given a weight vector of
shape `(40,)`. Instead of looping over 500 days:

```python
portfolio_returns = returns_matrix @ weights   # matrix-vector product, one line
```

**Where it's used in this project**

- `data/ingestion/feature_engineering.py::compute_beta()` — computes rolling
  beta using `np.cov(a[mask], b[mask])`, a vectorised covariance calculation
  over a rolling window rather than a manual sum-of-products loop.
- `models/rl_agent/portfolio_env.py::softmax()` — the entire softmax
  projection (`x - np.max(x)`, then `np.exp(x)`, then division) is three
  vectorised array operations, no loops, even though it's applied to a
  40-dimensional action vector on every single environment step (hundreds of
  thousands of times during training — this speed matters a lot).
- `models/rl_agent/portfolio_env.py::step()` — `np.dot(new_weights,
  asset_returns)` computes the portfolio's daily return as one dot product
  instead of summing `weight[i] * return[i]` in a Python loop across 40 assets.
- `models/classical/markowitz.py` — the entire mean-variance optimisation
  works on NumPy arrays (`expected_returns`, `cov_matrix`) that get fed
  directly into the `cvxpy` solver.

---

### 2. Pandas — DataFrames, Time Series Indexing, Resampling

**The core idea**

A Pandas `DataFrame` is a NumPy array with labels attached — each row has an
index (often a date) and each column has a name. This labelling is what makes
Pandas suited for time series: instead of remembering that "row 47" means
"March 3rd, 2021", you can just ask for `df.loc["2021-03-03"]`.

**DatetimeIndex**

When the index of a DataFrame is converted to actual datetime objects (not
strings), Pandas unlocks calendar-aware operations: slicing by date range,
resampling to a different frequency, and aligning two time series that don't
share the exact same dates.

```python
import pandas as pd

df.index = pd.to_datetime(df.index)   # turn string dates into real datetimes
df.index.name = "date"

# Now you can slice by date range directly
subset = df.loc["2022-01-01":"2022-06-30"]
```

**`.loc` vs `.iloc`**

- `.loc` selects by **label** (e.g. a date, or a column name).
- `.iloc` selects by **integer position** (e.g. "give me row 47", regardless
  of what its label is).

This distinction matters a lot in a walk-forward pipeline: you often need
"the last 504 rows before this date" (`.iloc`) as opposed to "everything
between these two calendar dates" (`.loc`).

**`.rolling()` — moving windows**

`.rolling(window=21)` creates a sliding window over the last N rows and lets
you apply an aggregation (mean, std, sum) to each window. This is the
backbone of almost every technical indicator in finance.

```python
returns = df["log_return"]
volatility_21d = returns.rolling(21).std() * (252 ** 0.5)   # annualised vol
momentum_63d   = returns.rolling(63).sum()                   # 3-month cumulative return
```

**`.resample()` — changing frequency**

`.resample("M")` groups daily data into monthly buckets so you can, for
example, turn daily returns into monthly returns by summing within each
month. This project uses this idea (manually, via slicing) when it
aggregates daily excess returns into monthly buckets for the significance
test.

**Aligning DataFrames with different trading calendars**

Indian markets (NSE) and US markets are closed on different holidays. If you
naively concatenate a Nifty50 stock's data with an S&P500 stock's data, you
will get rows with missing values on days one market was open and the other
was closed. The fix is to compute the **intersection** of both calendars —
only keep dates where *every* asset has data.

**Real-world example**

If RELIANCE.NS trades on a US holiday (say July 4th) but AAPL doesn't, and
you want to compute a joint portfolio return for that day, you literally
cannot — one leg of the portfolio has no price. So you drop that date from
your universe entirely rather than guessing a value.

**Where it's used in this project**

- `data/ingestion/fetch_data.py` — sets `df.index = pd.to_datetime(df.index)`
  right after download so every downstream operation can rely on a proper
  DatetimeIndex.
- `data/ingestion/feature_engineering.py` — `compute_volatility()` and
  `compute_momentum()` both use `.rolling(window)` to build features; the
  final line `features.dropna(subset=["momentum_126d", "beta_63d",
  "price_vs_ma200"])` removes the "warm-up" rows at the start of the series
  where a 200-day rolling average simply doesn't exist yet.
- `models/rl_agent/data_loader.py::load_aligned_features()` — this is the
  calendar-alignment problem solved directly: it loads every ticker's
  DataFrame, then computes `common_dates = idx if common_dates is None else
  common_dates.intersection(idx)` across all 40 tickers, so the RL
  environment only ever sees days where every single asset has a valid
  price. Without this step the environment would crash on the first missing
  value.
- `data/ingestion/macro_features.py` — uses `.ffill()` (forward-fill) to
  carry Friday's VIX value forward through the weekend, since VIX isn't
  quoted on days markets are closed.

---

### 3. scikit-learn — Preprocessing, Scalers, Covariance Estimators

**The core idea**

Neural networks and optimisers work much better when input features are on
similar numeric scales. A feature like "RSI" ranges roughly 0–100, while
"log_return" is typically a small number like 0.003. Without scaling, the
LSTM's gradient descent would effectively ignore the small-scale feature and
be dominated by the large-scale one.

**Why `RobustScaler` specifically (not `StandardScaler`)**

`StandardScaler` subtracts the mean and divides by standard deviation. The
problem: financial returns have fat tails — occasional extreme days (crashes,
earnings surprises) that are much larger than a typical day. A single crash
day can massively distort the mean and standard deviation, corrupting the
scaling for every *other*, ordinary day.

`RobustScaler` instead subtracts the **median** and divides by the
**interquartile range (IQR)** — the range between the 25th and 75th
percentile. Both of these statistics are far less sensitive to a handful of
extreme outliers, which is exactly the property you want with noisy
financial data.

```python
from sklearn.preprocessing import RobustScaler

scaler = RobustScaler()
scaler.fit(train_features)          # learns median & IQR from TRAINING data only
train_scaled = scaler.transform(train_features)
test_scaled  = scaler.transform(test_features)   # applies the SAME median/IQR
```

**Why you re-fit the scaler on training windows only**

This is one of the most important ideas in the whole project, so it's worth
repeating here and again under Data Leakage below: if you fit the scaler on
the *entire* dataset (train + test combined), the median and IQR calculated
would have "seen" statistics from the future test period. That's a subtle
form of cheating — your test-set performance would look better than it will
actually be in live trading, because the model was implicitly given
information about the future distribution of the data.

**Ledoit-Wolf covariance estimator**

Separately from scaling, scikit-learn also provides `LedoitWolf`, a
covariance matrix estimator (this is covered in depth under Tier 1, Portfolio
Theory, since it's fundamentally a finance concept — but the *tool* is a
scikit-learn class).

**Where it's used in this project**

- `models/forecasting/dataset.py::make_dataloaders()` — this is the textbook
  correct pattern:
  ```python
  scaler = RobustScaler()
  train_features = scaler.fit_transform(feature_data[:train_end])   # fit + transform on train
  val_features   = scaler.transform(feature_data[train_end:val_end]) # transform only
  test_features  = scaler.transform(feature_data[val_end:])          # transform only
  ```
  Notice `fit_transform` is called *only* on the training slice; `val` and
  `test` only ever call `.transform()`, reusing statistics learned from the
  past.
- `models/classical/markowitz.py::ledoit_wolf_cov()` — wraps
  `sklearn.covariance.LedoitWolf().fit(returns)` to produce a shrunk,
  numerically stable covariance matrix for the optimiser (details under
  Tier 1).

---

### 4. Parquet File Format

**The core idea**

Parquet is a **columnar** binary file format, as opposed to CSV, which is a
**row-oriented** text format. In a row-oriented format, all the values of a
single row sit next to each other on disk. In a columnar format, all the
values of a single *column* sit next to each other instead.

**Why columnar storage is faster for analytics**

Most analytical queries only touch a handful of columns out of many (e.g.
"give me just the `close` and `log_return` columns for RELIANCE.NS"). A
row-oriented format like CSV has to read every single row from disk in full,
including columns you don't need, because rows are stored as contiguous
blocks of *all* fields. A columnar format can skip straight to the specific
columns requested and read only those bytes.

Columnar storage also compresses better: a column of dates, or a column of
floating-point returns, has much more internal repetition and predictability
than a mixed row of different data types, so compression algorithms shrink
it further.

**Concrete comparison**

| | CSV | Parquet |
|---|---|---|
| Storage | Row-oriented text | Column-oriented binary |
| Compression | Poor (or none) | Strong (columns compress well) |
| Reading a subset of columns | Reads the whole file | Reads only needed columns |
| Data types preserved | No — everything is text, re-parsed on load | Yes — types are stored in the schema |
| Typical size | 1x (baseline) | ~4–10x smaller |

**DVC (Data Version Control) and why it doesn't store Parquet files in Git**

Git is designed for tracking changes to small text files (source code). A
40-ticker, 5-year OHLCV dataset in Parquet can easily be tens or hundreds of
megabytes, and every retrain would create a new version — Git repos are not
built to store large binary blobs efficiently; every clone of the repo would
have to download the *entire history* of every version of every file. DVC
solves this by storing a small `.dvc` pointer file in Git (just a hash and a
path) while the actual data lives in external storage (S3, Google Drive,
etc.). Git tracks *which version* of the data you're using; DVC tracks
*where to fetch it from*.

**Where it's used in this project**

- Every ingestion and feature engineering step reads/writes Parquet:
  `data/ingestion/fetch_data.py` saves raw OHLCV as
  `data/raw/{ticker}.parquet`; `feature_engineering.py` saves engineered
  features as `data/processed/features_{ticker}.parquet`.
- `data/raw.dvc` is the DVC pointer file that tracks the `data/raw/` folder
  without committing the actual price data into Git history.
- `backtesting/backtest_results.json` is deliberately **not** Parquet, since
  it's a small, human-readable summary meant to be consumed directly by the
  dashboard and report generator — Parquet's advantages only kick in at
  larger, columnar, read-heavy datasets.

---

### 5. Train / Validation / Test Split — Why Order Matters in Time Series

**The core idea**

In a typical (non-time-series) ML problem — say, classifying images of cats
vs dogs — every training example is independent of every other example. You
can shuffle the dataset randomly and split it into train/val/test however
you like, because there's no meaningful "order" to the data.

Time series data is fundamentally different: **each row depends on the rows
before it**, and critically, **the future should never influence the past**.
If you shuffle a time series and randomly put, say, March 2023 in the
training set and February 2023 in the test set, your model implicitly gets
to "see the future" relative to some of its training examples — the model
could learn patterns from March and apply them to "predict" February, which
is impossible in real trading (you can never know March's data before
February happens).

**The correct approach: strict temporal ordering**

```
|------------------- Train (70%) -------------------|--- Val (15%) ---|--- Test (15%) ---|
     earliest dates                                                        latest dates
```

Every split boundary is a specific date. Training data is strictly earlier
than validation data, which is strictly earlier than test data. No shuffling
across these boundaries is allowed — although *within* the training set, you
are still allowed to shuffle the order that batches are fed to the model
during training (this doesn't leak information, since all training examples
are already from the past relative to validation/test).

**Real-world analogy**

Imagine you're a weather forecaster being graded on your predictions. It
would be absurd to grade your April 2024 forecasts using information you
only had access to in June 2024. The test of a real forecasting model is:
"given only what was known at the time, how good was the prediction?" Time
series ML must respect the same constraint.

**Why interviewers specifically test for this**

It's the single most common way beginners accidentally inflate their
reported model performance without realizing it. A model with shuffled
train/test splits on financial data will often report suspiciously great
backtest metrics that completely fall apart in live trading — because the
backtest wasn't really testing prediction, it was testing memorisation of
a period the model had (invisibly) already been trained on parts of.

**Where it's used in this project**

- `models/forecasting/dataset.py::make_dataloaders()` computes
  `train_end = int(n * train_frac)` and `val_end = int(n * (train_frac +
  val_frac))` and slices the DataFrame using plain positional ranges
  (`feature_data[:train_end]`, `feature_data[train_end:val_end]`,
  `feature_data[val_end:]`) — never a random shuffle across these
  boundaries. Only `DataLoader(train_ds, shuffle=True)` shuffles anything,
  and that's just the order of already-past training examples fed to the
  optimiser, which is fine.
- `backtesting/walk_forward.py` takes this even further with a **rolling**
  version of the same idea — instead of one fixed split, it repeatedly
  slides a training window (504 days) and a test window (126 days) forward
  through time, always keeping test data strictly after training data for
  that specific window (`TRAIN_DAYS`, `TEST_DAYS`, `STEP_DAYS` constants).

---

### 6. Data Leakage — What It Is and How to Prevent It

**The core idea**

Data leakage happens whenever information that would not actually be
available at prediction time sneaks into the training process — whether
that's future prices, future statistics, or even statistics computed from a
combination of past *and* future data.

It is easy to introduce accidentally because the leak often doesn't look
like an obvious mistake — it can hide inside a seemingly innocent
preprocessing step.

**The three concrete leakage traps this project explicitly guards against**

1. **Global normalisation statistics.** If you compute the mean/std (or
   median/IQR) of a feature across the *entire* dataset before splitting
   into train/test, the "past" model has implicitly learned something about
   the future distribution of that feature — because the scaler's
   parameters were influenced by future data points. **Fix:** fit the scaler
   only on the training window (`dataset.py`, as detailed under scikit-learn
   above).

2. **Covariance matrix computed with look-forward data.** In the Markowitz
   optimiser, if you calculated the covariance matrix using returns from
   both before *and* after the rebalancing date, you'd be handing the
   optimiser information about how assets will actually co-move in the
   future — which is precisely what a real portfolio manager never has.
   **Fix:** `MarkowitzOptimiser.simulate()` in `models/classical/markowitz.py`
   only ever passes `return_matrix[t - lookback_days : t]` — a strictly
   backward-looking window — into `compute_weights()`.

3. **Look-ahead bias in backtesting more generally.** Even at the level of
   the overall experiment design, if a model is retrained "just once" on
   the full historical dataset and then tested on a slice of that same
   dataset, its true out-of-sample skill is unknown. **Fix:** the whole
   `walk_forward.py` methodology exists specifically to eliminate this —
   discussed in depth under Tier 1's Walk-Forward Backtesting section.

**A subtle leakage trap worth knowing (interview-favourite)**

Suppose you engineer a "20-day moving average" feature and, at row `t`, you
accidentally compute it using `df["close"].rolling(20, center=True).mean()`
instead of a normal (trailing) rolling window. The `center=True` option
makes Pandas center the window around row `t`, meaning it uses **10 days
into the future** as part of the "current" feature. This is a very easy
mistake to make and produces a feature that looks completely reasonable but
is quietly leaking the future.

**Where it's used in this project**

Every rolling computation in `feature_engineering.py`
(`compute_momentum`, `compute_volatility`, `compute_beta`, `compute_rsi`,
`price_vs_ma50`, `price_vs_ma200`) uses the *default*, trailing
`.rolling(window)` behaviour — the window looks backward from row `t`, never
forward — which is the correct, leak-free way to compute a technical
indicator.

---

### 7. Log Returns vs Price Returns

**The core idea**

There are two common ways to express "how much did this asset move":

- **Simple (price / arithmetic) return:** `(P_t - P_{t-1}) / P_{t-1}`
- **Log return:** `ln(P_t / P_{t-1})`

At first glance these look almost identical for small daily moves (a 1% move
is roughly 0.00995 in log terms — very close). But the differences become
important as soon as you start aggregating returns over time or feeding them
into a neural network.

**Why log returns are additive over time**

If a stock returns `r1` on day 1 and `r2` on day 2, the *simple* two-day
return is **not** `r1 + r2` — it's `(1+r1)(1+r2) - 1`, a multiplicative
compounding formula. Log returns, by contrast, genuinely add:

```
log_return(day1) + log_return(day2) = ln(P1/P0) + ln(P2/P1) = ln(P2/P0)
                                     = log_return(2-day period)
```

This means you can sum a series of daily log returns to get the total log
return over any period — exactly what `momentum_21d = returns.rolling(21).sum()`
relies on in this project's feature engineering.

**Why log returns are (closer to) normally distributed**

Simple returns are bounded below at -100% (you can't lose more than
everything) but unbounded above (a stock can 10x). This creates a
right-skewed distribution. Taking the log return removes this floor
constraint (`ln` maps `(0, ∞)` to `(-∞, ∞)`), producing a distribution that's
much closer to symmetric/normal — a helpful property for models (like the
LSTM here) that implicitly assume roughly Gaussian-ish residuals, and for
statistical tests (like the t-test used for significance testing) that
assume approximate normality.

**Why log returns are scale-invariant**

A log return doesn't care whether the stock is priced at ₹10 or ₹10,000 — a
1% daily move is `ln(1.01) ≈ 0.00995` regardless of the absolute price
level. This makes it possible to compare returns across a ₹500 Indian stock
and a $500 US stock on equal footing, which matters a great deal in this
project since it mixes Nifty50 and S&P500 assets in one universe.

**Worked example**

Stock price goes from ₹100 → ₹110 → ₹100 (up 10%, then back down).

- Simple returns: `+10%`, then `-9.09%`. Sum ≠ 0% (compounding effect).
- Log returns: `ln(110/100) = +0.0953`, then `ln(100/110) = -0.0953`. Sum
  = exactly `0` — correctly reflecting that the price ended up unchanged.

**Where it's used in this project**

- `data/ingestion/feature_engineering.py::compute_returns()`:
  `np.log(df["Close"] / df["Close"].shift(1))` — this single line is the
  entire returns computation, and every downstream feature (momentum,
  volatility, RSI) is built on top of it.
- `models/rl_agent/portfolio_env.py::step()` — the portfolio's per-step
  return is `np.dot(new_weights, asset_returns)`, a weighted sum of log
  returns, and the portfolio value is updated by
  `self._portfolio_value *= np.exp(portfolio_return)` — exponentiating a log
  return is exactly how you convert it back into a growth multiplier.
- `backtesting/metrics.py::equity_curve()` — `initial * np.exp(np.cumsum(returns))`
  builds the entire equity curve by cumulatively summing log returns and
  then exponentiating once at the end, relying directly on the additive
  property described above.

---

## TIER 1 — Finance Fundamentals

### 1. Sharpe Ratio

**The core idea**

The Sharpe ratio measures **return earned per unit of total risk taken**. A
strategy that returns 15% a year by swinging wildly up and down is, by this
measure, worse than a strategy that returns 12% a year very smoothly.

**Formula**

```
Sharpe = (mean_return − risk_free_rate) / std_dev_of_returns  ×  √252
```

The `√252` factor **annualises** a daily Sharpe ratio (252 is the typical
number of trading days in a year). Since standard deviation scales with the
square root of time (a statistical property of random walks) while mean
return scales linearly with time, the ratio of the two scales with `√T`.

**Intuition for the numbers**

- Sharpe < 0: losing money on a risk-adjusted basis.
- Sharpe 0–1: mediocre — you're taking on risk without much reward.
- Sharpe 1–2: good, professionally respectable.
- Sharpe > 2: excellent — rare to sustain over long periods.

**Why volatility, and not just total return, matters**

Two funds might both return 10% this year. Fund A did it by going up
steadily every month. Fund B did it by losing 40% in March and gaining it
all back plus more by December. Fund B is objectively riskier to hold — an
investor who needed to withdraw money in March would have been forced to
realise a huge loss. The Sharpe ratio captures this difference even though
both funds show the same headline return.

**Where it's used in this project**

- `backtesting/metrics.py::sharpe_ratio()` is the primary metric used
  throughout — in `walk_forward.py`, in the champion-challenger promotion
  logic (`train_agent.py` compares `val_sharpe` against the current
  champion), and as the headline KPI on the Streamlit dashboard's
  Performance page.
- Reported results in this project: RL agent Sharpe ≈ **1.4–1.47**, versus
  Markowitz ≈ 0.74–0.77 and the Nifty50 benchmark ≈ 0.61.

---

### 2. Sortino Ratio

**The core idea**

The Sortino ratio is a variant of the Sharpe ratio that only penalises
**downside** volatility (days where returns are negative), ignoring upside
volatility entirely.

**Why this matters — the key flaw in the Sharpe ratio it fixes**

Sharpe treats all volatility as bad — a huge positive day (say, +8% in one
session) increases the standard deviation just as much as a huge negative
day (-8%) would, and gets penalised identically. But investors don't
actually dislike large *positive* surprises — nobody complains about a
strategy that occasionally makes a lot of money in a single day. Sortino
fixes this by only measuring the spread of the *negative* returns.

**Formula**

```
downside_std = sqrt(mean(negative_excess_returns²))
Sortino = (mean_return − risk_free_rate) / downside_std  ×  √252
```

Only returns below the risk-free rate (in this project, simplified to 0)
contribute to `downside_std`; positive returns are excluded from the
denominator entirely.

**Real-world example**

Suppose over 10 days a strategy has returns:
`+5%, +6%, -1%, +4%, -0.5%, +7%, +3%, -1%, +2%, +8%`

Sharpe would treat the `+8%` day as "risky" and penalise the ratio for it,
even though that day was pure upside. Sortino only looks at `-1%, -0.5%,
-1%` when computing the risk denominator — a much smaller, more relevant
number — producing a materially higher (better) ratio for a strategy that
has occasional big up-days.

**Why this project uses Sortino as the RL reward function specifically**

An RL agent is trained by directly maximising its reward signal. If you use
Sharpe as the reward, the agent is mathematically incentivised to *avoid
large positive returns*, because they increase the denominator (total
volatility) and shrink the ratio — an obviously perverse outcome for a
trading strategy. Sortino removes this perverse incentive: the agent can
freely let winning trades run without being "punished" by its own reward
function for doing so, while still being penalised for choppy, loss-heavy
behaviour.

**Where it's used in this project**

- `models/rl_agent/reward.py::sortino_reward()` implements the
  episode-level Sortino calculation with the exact downside-only logic
  above, used for evaluation/logging of episode quality.
- `backtesting/metrics.py::sortino_ratio()` is the backtest-level
  implementation, reported alongside Sharpe in every strategy comparison.
- Documented improvement: switching the reward signal from Sharpe-style to
  Sortino-style reportedly produced **40% more stable training** and a
  **+0.3 Sharpe improvement** on validation, according to the project's
  README.
- **Nuance worth knowing for interviews:** the environment's actual
  per-step training signal (`models/rl_agent/portfolio_env.py::step()` →
  `reward.py::step_reward()`) is simpler than full Sortino — it's just
  `log_return − transaction_cost_penalty` computed fresh at every single
  timestep. This is because PPO's per-step advantage estimation works
  better with a dense, immediate reward signal rather than a single Sortino
  number computed only at the end of a 252-day episode. The **episode-level
  Sortino** (`sortino_reward()`) is instead used at evaluation time to
  *score* how good a finished episode was, and Sortino/Sharpe are the
  headline metrics reported in `evaluate_agent.py` and MLflow — so Sortino
  shapes *how the agent is judged and promoted*, even though the *moment-to-
  moment* training signal is the simpler log-return-minus-cost formula.

---

### 3. Maximum Drawdown

**The core idea**

Maximum drawdown (Max DD) answers a very human question: "at the absolute
worst point, how much of my money would I have lost from my previous peak,
before it recovered?" It's measured as the largest peak-to-trough decline in
portfolio value over the whole period being examined.

**Formula**

```
equity_t = running portfolio value at time t
peak_t   = the highest equity value reached up to time t (running maximum)
drawdown_t = (equity_t − peak_t) / peak_t     (always ≤ 0)
Max Drawdown = min(drawdown_t) over the whole period
```

**Worked example**

A portfolio starts at ₹100, climbs to ₹150, falls to ₹90, then recovers to
₹200.

- Peak before the fall: ₹150.
- Trough: ₹90.
- Drawdown at the trough: `(90 - 150) / 150 = -40%`.
- Even though the portfolio eventually reached a *new* all-time high of
  ₹200, the Max Drawdown for the whole period is still **-40%** — the
  worst-case pain an investor holding through that period would have felt.

**Why it's a critical risk metric even when Sharpe looks great**

Imagine a strategy with an excellent Sharpe ratio of 1.8 but a max drawdown
of -55%. On paper, the *average* risk-adjusted return looks strong. But an
investor who happened to need their money during that -55% drawdown would
have suffered devastating, possibly unrecoverable losses. Many investors
(and fund mandates) have hard limits on acceptable drawdown regardless of
how good the Sharpe ratio looks — a strategy can be statistically excellent
on average and still be practically unusable if its worst-case behaviour is
too extreme.

**Where it's used in this project**

- `backtesting/metrics.py::max_drawdown()` computes this via
  `equity = np.exp(np.cumsum(returns))`, then `peak =
  np.maximum.accumulate(equity)` (a running maximum), then `dd = (equity -
  peak) / peak`, taking the minimum.
- `backtesting/metrics.py::drawdown_series()` produces the *entire time
  series* of drawdown values (not just the single worst number), which
  powers the drawdown chart on the Streamlit dashboard's Performance page —
  letting you visually see not just how deep the worst drawdown was, but how
  *long* the portfolio spent underwater before recovering.
- Reported result: RL agent Max Drawdown ≈ **-16% to -33%** (two different
  backtest runs in the README/SYSTEM_DESIGN report slightly different
  numbers), versus the benchmark's -22.8% to -41.5% — the RL strategy loses
  less in its worst moments than simply holding the index or equal-weighting.

---

### 4. Value at Risk (VaR) and Conditional VaR (CVaR)

**The core idea**

VaR and CVaR both try to answer: "on a *typical bad* day, how much could I
realistically lose?" — but they answer slightly different versions of that
question, and CVaR is considered the more conservative, more honest answer.

**Value at Risk (VaR)**

VaR at the 95% confidence level is the daily loss such that, historically,
95% of days were *better* than this and only 5% of days were *worse*. It's
simply the 5th percentile of the historical daily return distribution.

```python
VaR_95 = 5th percentile of daily returns
```

**Worked example**

If a portfolio's daily returns over the past year, sorted from worst to
best, show that the 5th percentile is `-2.1%`, that means: "on 95% of days,
I lost less than 2.1%; on the worst 5% of days, I lost 2.1% or more."

**The problem with VaR — it says nothing about *how bad* the tail gets**

VaR tells you the *threshold* of a bad day but says absolutely nothing about
what happens beyond that threshold. Two portfolios could have an identical
VaR of -2.1%, but one of them might have its worst days capped around -3%,
while the other has occasional catastrophic days of -15% or worse. VaR alone
cannot distinguish between these — which is exactly why risk managers prefer
CVaR.

**Conditional VaR (CVaR) / Expected Shortfall**

CVaR answers: "given that I'm already in that worst 5% of days, what's the
*average* loss I should expect?" It's computed by taking the mean of all
returns that fall at or below the VaR threshold.

```python
CVaR_95 = mean(all returns ≤ VaR_95)
```

**Worked example continued**

If the worst 5% of days (below the -2.1% VaR threshold) had actual returns
of `-2.2%, -2.5%, -3.1%, -4.8%, -9.7%`, the CVaR is the average of *those*
numbers — a much more informative, and worse (more negative), number than
the VaR threshold alone. CVaR captures the "how bad can it really get"
question that VaR is blind to.

**Why CVaR is preferred by risk managers**

CVaR is a *coherent* risk measure in the technical/mathematical sense (it
satisfies subadditivity — the risk of a combined portfolio is never
underestimated relative to its parts), while VaR is not. In plain terms:
CVaR can't be "gamed" by structuring a portfolio to hide catastrophic tail
risk just outside the VaR cutoff, because CVaR explicitly measures what's
happening in that tail.

**Where it's used in this project**

- `backtesting/metrics.py::value_at_risk()` and `::conditional_var()`
  implement exactly the percentile-then-average logic described above.
- Both are reported side-by-side in every strategy comparison table (dashboard
  Comparison page, `compute_all_metrics()`), with CVaR 95% for the RL agent
  reported around **-1.8% to -2.7%** depending on the backtest run — meaning
  on the worst 5% of days, the RL portfolio's average loss stayed
  meaningfully better contained than the benchmarks' -3.2% to -3.8%.

---

### 5. CAGR — Compound Annual Growth Rate

**The core idea**

CAGR answers: "if this portfolio had grown at a perfectly smooth, constant
annual rate the entire time, what would that rate have to be to produce the
same total return I actually got?" It converts a messy, volatile multi-year
return path into a single, comparable "as if it were smooth" annual number.

**Why you can't just average the yearly returns**

Suppose a portfolio returns +50% in year 1 and -50% in year 2. The simple
average of those two numbers is 0%, suggesting "no change" — but that's
wrong. Starting with ₹100: after year 1 you have ₹150; after year 2 you lose
50% of ₹150, leaving ₹75. You actually lost 25% overall, not 0%. CAGR
correctly accounts for **compounding**, where gains and losses build on top
of each other multiplicatively, not additively.

**Formula (from log returns, as used in this project)**

```
total_log_return = sum of all daily log returns over the period
n_years           = number of trading days / 252
CAGR = exp(total_log_return / n_years) − 1
```

This works because of the additive property of log returns discussed in
Tier 0: summing all the daily log returns gives you the total log return for
the whole period, dividing by the number of years annualises it, and
`exp()` converts back from log-return space into an actual growth-rate
percentage.

**Worked example**

Using the same two-year scenario: ₹100 → ₹150 → ₹75 is a total return of
`75/100 - 1 = -25%` over 2 years. The CAGR would be
`(75/100)^(1/2) - 1 ≈ -13.4%` per year — i.e. "an equivalent constant -13.4%
per year, for two years, would have taken you from ₹100 to ₹75," which is a
far more honest single summary number than the misleading simple average of
0%.

**Where it's used in this project**

- `backtesting/metrics.py::annualised_return()` (this is the function that
  actually computes CAGR, despite the more generic name) and
  `models/rl_agent/evaluate_agent.py::compute_metrics()` both implement the
  exact log-return-based formula above.
- Reported results: RL agent CAGR ≈ **14.3%–20%** across different backtest
  runs, versus the benchmark's 8.7% and equal-weight's 9.4%–16.9%.

---

### 6. Markowitz Mean-Variance Optimisation (MVO)

**The core idea**

Given a set of assets, each with some expected return and some risk
(volatility, and how they move together), Markowitz's 1952 framework finds
the portfolio weights that deliver the **highest expected return for a given
level of risk** — or equivalently, the **lowest risk for a given level of
return.**

**The two required inputs**

1. An **expected returns vector** — one number per asset, "what do I expect
   this asset to return on average."
2. A **covariance matrix** — how every pair of assets moves together. This
   is what allows the optimiser to find *diversification benefits*: even if
   two assets individually look risky, if they tend to move in *opposite*
   directions, combining them can reduce the overall portfolio's risk below
   what either asset has alone.

**Why this is a convex optimisation problem**

The objective (minimise portfolio variance, or maximise Sharpe) combined
with the constraints (weights sum to 1, weights are non-negative, weights
are bounded) forms a well-behaved mathematical problem with a single global
optimum — no risk of getting stuck in a "locally good but not actually best"
solution, unlike many machine learning optimisation problems. This is
exactly the kind of problem the `cvxpy` library (a convex optimisation
modelling language) is designed to solve efficiently and reliably.

**Concrete example of diversification**

Suppose Asset A (an airline stock) does badly when oil prices spike, and
Asset B (an oil company stock) does *well* when oil prices spike. Held
individually, both are volatile and exposed to oil-price risk. Held
*together* in the right proportions, the oil-price risk partially cancels
out — the portfolio's overall variance can be lower than either asset's
variance alone. Markowitz's covariance matrix is precisely what allows the
optimiser to discover and exploit this kind of relationship mathematically,
rather than by intuition.

**Why this project caps the maximum weight per asset at 40%**

Without a cap, the unconstrained "maximum Sharpe" solution can sometimes
recommend putting almost all capital into just one or two assets that
happened to look statistically great in the historical sample — a
dangerously concentrated, overfit-prone portfolio. Capping any single
asset's weight at 40% forces the optimiser to keep some minimum level of
diversification even if the raw numbers suggest otherwise.

**Where it's used in this project**

- `models/classical/markowitz.py::markowitz_weights()` sets up exactly this
  convex problem:
  ```python
  constraints = [cp.sum(w) == 1, w >= 0, w <= max_weight]
  objective = cp.Minimize(port_var)
  constraints.append(port_return >= target_return)
  prob.solve(solver=cp.ECOS)
  ```
  This is the "minimise variance for a target return" formulation of
  Markowitz, mathematically equivalent to maximising Sharpe at that return
  level.
- It's used as the **primary baseline** the RL agent is compared against
  throughout the project — in `walk_forward.py`, `baselines.py`,
  `compare_all.py`, and the dashboard's Comparison and Frontier pages.
- Solver failures are handled gracefully: if `cvxpy` can't find a solution
  (numerical issues, infeasible constraints), the code falls back to
  `np.ones(n) / n` — plain equal-weighting — rather than crashing, logging a
  warning instead.

---

### 7. Efficient Frontier

**The core idea**

If you plot every *possible* portfolio (every combination of weights across
your assets) on a chart with risk (volatility) on the x-axis and expected
return on the y-axis, you get a cloud of points. The **efficient frontier**
is the upper-left boundary of that cloud — the set of portfolios that
deliver the *maximum possible return for their level of risk* (or
equivalently, the *minimum possible risk for their level of return*).

**Why every point on the frontier is "Pareto-optimal"**

"Pareto-optimal" means you cannot improve one thing without making another
thing worse. For a portfolio *on* the frontier, there is no other
achievable portfolio with the same risk that has higher return — and no
other achievable portfolio with the same return that has lower risk. Any
portfolio sitting *below* the frontier is provably suboptimal: there exists
some other combination of the same assets that does strictly better on at
least one axis without doing worse on the other, so a rational investor
should never hold a below-frontier portfolio.

**Real-world intuition**

Imagine shopping for a used car and plotting every listing by price (risk)
vs mileage-adjusted condition (return). The "best deals" trace out a curve —
for any given price, there's a best possible condition you can get; paying
more than that price for the same condition is objectively a worse deal.
The efficient frontier is the financial-portfolio equivalent of that "best
deals" curve.

**Where it's used in this project**

- `dashboard/pages/4_Frontier.py` visualises this using a **Monte Carlo
  simulation** approach (rather than solving for the frontier analytically):
  it generates thousands of *random* portfolios (`np.random.dirichlet`,
  which produces random weight vectors that automatically sum to 1),
  computes each one's return and volatility, and plots them as a scatter
  cloud. The upper-left edge of that cloud visually approximates the true
  efficient frontier. The single best (max-Sharpe) random portfolio found is
  highlighted with a gold star, and the RL agent's own actual
  return/volatility point is plotted as a blue diamond for direct visual
  comparison against the frontier — letting a viewer see, at a glance,
  whether the RL agent is producing a risk-adjusted return that's
  competitive with (or even better than) what a huge random search over
  static allocations could find.

---

### 8. Ledoit-Wolf Covariance Shrinkage

**The core idea**

Markowitz optimisation needs a covariance matrix as an input. The most
obvious way to estimate one is the **sample covariance matrix** — just
calculate it directly from historical returns. The problem: with a limited
number of historical observations (say, 504 trading days) and a large number
of assets (40, in this project), the sample covariance matrix becomes
**noisy and poorly conditioned** — meaning small differences in the
historical sample can cause wildly different, unstable optimizer outputs.

**Why "poorly conditioned" is a real problem, not just a theoretical one**

A covariance matrix needs to be inverted (or used in ways mathematically
equivalent to inversion) inside the optimisation. When the number of assets
(N=40) is large relative to the number of observations (T=504), the sample
covariance matrix can become close to *singular* (not invertible, or
numerically unstable to invert) — small numerical errors get massively
amplified, and the resulting "optimal" weights can be extreme and
nonsensical (e.g. suggesting -300% in one asset and +400% in another to
exploit a statistically noisy, not real, correlation pattern).

**The Ledoit-Wolf fix — shrinkage**

Ledoit-Wolf shrinkage blends the noisy sample covariance matrix with a
simpler, "structured" target matrix (conceptually, something close to a
scaled identity matrix) that is guaranteed to be well-behaved and
invertible:

```
Σ_shrunk = (1 − α) × Σ_sample + α × μ × I
```

Here `α` (the shrinkage intensity) is estimated *analytically* — directly
from a formula derived by Ledoit and Wolf — rather than by expensive
cross-validation. The result is a matrix that keeps most of the genuine
signal from the sample covariance while "pulling in" the noisiest,
least-reliable parts toward a stable, simple target.

**Analogy**

Think of shrinkage like a weather forecaster who has only 5 days of local
data for a town but decades of data for the wider region. Rather than
trusting the noisy 5-day local average completely, or ignoring it
completely in favour of the regional average, a smart forecaster blends the
two — leaning more on the noisy local data where it's genuinely informative,
and more on the stable regional average where the local sample is too small
to trust on its own.

**Where it's used in this project**

- `models/classical/markowitz.py::ledoit_wolf_cov()` wraps
  `sklearn.covariance.LedoitWolf().fit(returns).covariance_` — a one-line
  call to scikit-learn's implementation.
- It's the covariance estimate fed into every single Markowitz optimisation
  call throughout the project (`MarkowitzOptimiser.compute_weights()`,
  `dashboard/pages/4_Frontier.py`), always recomputed from a strictly
  backward-looking lookback window (never the full dataset) to avoid the
  look-ahead bias discussed in Tier 0.

---

### 9. Transaction Costs and Portfolio Turnover

**The core idea**

Every real trade costs something — the bid-ask spread, brokerage fees, and
market impact (the act of trading itself can move the price against you,
especially for larger orders). A backtest that ignores transaction costs is
implicitly assuming a fictional, frictionless market, and will overstate how
profitable a strategy really is — especially for strategies that trade
frequently.

**Portfolio turnover**

Turnover measures how much of the portfolio's composition changed at a
rebalancing event — it's the sum of the absolute value of every weight
change:

```
turnover = sum(|weight_new_i − weight_old_i|)  for every asset i
```

**Worked example**

Suppose a portfolio holds 50% Asset A and 50% Asset B, and rebalances to
30% A / 70% B. Turnover = `|0.30 - 0.50| + |0.70 - 0.50| = 0.20 + 0.20 =
0.40` — meaning 40% of the portfolio's value was effectively "traded" during
that rebalance (even though the weights only moved by 20 percentage points
each, both moves count toward turnover since they represent actual buy/sell
activity).

**Modelling transaction cost as a penalty**

This project charges `0.1%` of turnover as a cost at every rebalancing
event:

```python
tc_penalty = TRANSACTION_COST_RATE * turnover     # TRANSACTION_COST_RATE = 0.001
```

**Why this matters enormously for an RL agent specifically**

An RL agent, left unconstrained, has no inherent notion that trading is
costly — if the reward function is just "portfolio return," the agent might
learn to chase every tiny, noisy signal by reallocating dramatically every
single day, racking up trading costs that would completely erode real-world
profitability, even though the *backtest* (which might not model costs)
would show great performance. By baking the transaction cost penalty
directly into the reward signal, the agent is trained from the very start to
only make a trade when the expected benefit of that trade outweighs its
cost — the same tradeoff a real trader has to make.

**Real-world example**

Imagine two traders with identical stock-picking skill. Trader A rebalances
their portfolio every single day, chasing every tiny fluctuation. Trader B
only rebalances when there's a meaningful, high-conviction signal. Even if
both traders are equally good at *predicting* price direction, Trader A will
underperform Trader B in practice purely because trading costs eat away
their gains far more aggressively.

**Where it's used in this project**

- `models/rl_agent/reward.py` defines `TRANSACTION_COST_RATE = 0.001` (0.1%)
  and both `step_reward()` and `sortino_reward()` subtract a turnover-based
  penalty from the raw return.
- `models/rl_agent/portfolio_env.py::step()` passes `weights_prev` and
  `weights_curr` into the reward function on every single step, so the cost
  is charged continuously throughout an episode, not just at arbitrary
  rebalancing dates.
- `models/classical/markowitz.py::MarkowitzOptimiser.simulate()` rebalances
  only every 21 trading days (monthly) rather than daily — a design choice
  that implicitly limits turnover (and therefore transaction costs) for the
  classical baseline, though notably this baseline doesn't explicitly
  penalise turnover in its objective the way the RL reward does.

---

### 10. Walk-Forward Backtesting vs. a Single Train-Test Split

**The core idea**

A single train-test split tests your strategy on exactly one contiguous
slice of history — meaning exactly one "market regime" (e.g. a five-year
bull market, or a single crash-and-recovery period). A strategy can look
fantastic on one regime and fail completely on a different one (say, a
prolonged sideways/choppy market), and a single split would never reveal
that weakness.

**Walk-forward backtesting** fixes this by repeatedly rolling a fixed-size
training window and a fixed-size test window forward through time, so the
strategy gets evaluated across *many different, non-overlapping regimes*
within the available history.

**How this project's specific windowing works**

```
Train window:  504 trading days  (≈ 2 years)
Test window:   126 trading days  (≈ 6 months)
Step size:      21 trading days  (≈ 1 month) forward
```

```
Window 1:  [Train: day 0–503]   [Test: day 504–629]
Window 2:  [Train: day 21–524]  [Test: day 525–650]
Window 3:  [Train: day 42–545]  [Test: day 546–671]
...and so on, sliding forward by 21 days each time.
```

Each window's test period is a completely fresh 6-month slice of the
market that the model/strategy has never been evaluated on before — and
crucially, **the model is never retrained mid-test-window**, and **feature
scalers/covariance matrices are recomputed fresh from each window's own
training data**, so no window can "leak" information from a future window
into an earlier one.

**Why this is the gold standard for financial ML evaluation**

By aggregating performance across many independent windows spanning
multiple years, you get a much more honest picture of how consistent a
strategy really is — not just "did it get lucky on one favourable period,"
but "does it hold up across bull markets, bear markets, high-volatility
periods, and calm periods alike." This project's walk-forward engine
produces one Sharpe/Sortino/CAGR/drawdown number *per window*, which is then
displayed as a bar chart on the dashboard — letting you visually spot
whether the strategy's edge is consistent across time or concentrated in
just a couple of lucky windows.

**Connecting it to statistical significance**

Beyond visual inspection, this project also runs a formal **two-sided
t-test** comparing the RL strategy's monthly excess returns (RL return minus
equal-weight return, aggregated into monthly buckets across all windows)
against a null hypothesis of "the true average excess return is zero." A
resulting `p-value < 0.05` means: if the RL strategy actually had *no* real
edge over equal-weighting, there would be less than a 5% chance of observing
outperformance this large and this consistent purely by random luck.

**Where it's used in this project**

- `backtesting/walk_forward.py` implements exactly this rolling window
  scheme via the constants `TRAIN_DAYS = 504`, `TEST_DAYS = 126`, `STEP_DAYS
  = 21`, generating anywhere from ~11 to ~30 windows depending on the total
  history available (the README and SYSTEM_DESIGN report slightly different
  window counts from different runs).
- `backtesting/metrics.py::ttest_excess_returns()` implements the monthly
  aggregation + `scipy.stats.ttest_1samp` significance test described above.
- Reported result: `p ≈ 0.024` (README) or `p ≈ 4.3 × 10⁻²²` (SYSTEM_DESIGN,
  a different/larger run) — both comfortably below the 0.05 significance
  threshold, supporting the claim that the RL agent's outperformance versus
  equal-weight is not simply due to chance.

---

## Bonus Concept — `async` / `await` (needed for later, worth learning now)

You asked for this concept included early even though it isn't a Tier 0/1
topic on its own — it becomes essential once we reach the Backend & Serving
tier (FastAPI), so it's worth building the intuition now.

**The core idea**

Normally, when a program calls a function that has to *wait* for something
external — reading a file from disk, querying a database, calling another
server over the network — the entire program **blocks**: it sits idle doing
absolutely nothing until that slow operation finishes, even though the CPU
itself isn't doing any real work during the wait (it's just waiting on I/O).

`async`/`await` lets a single-threaded program **pause** a function at the
exact point it's waiting on something slow, hand control back to an event
loop, let that event loop go do other useful work (like starting to handle
a *different* incoming request) in the meantime, and then **resume** the
paused function automatically once the slow operation finally completes.

**Why this is different from just "running things in parallel with threads"**

Threads achieve concurrency by literally running multiple sequences of
instructions in parallel (or the OS rapidly switching between them), which
has real overhead (memory per thread, context-switching cost, and the risk
of subtle race conditions if two threads touch shared data at the same
time). `async`/`await` achieves concurrency *cooperatively*, within a
**single thread** — there is only one thing actually executing at any given
instant, but that one thing intelligently switches to other pending work
whenever it would otherwise be sitting idle waiting on I/O. This makes it
extremely efficient for **I/O-bound** workloads (lots of waiting on network
calls) specifically — though it does *not* help with **CPU-bound** work
(heavy computation), since there's no idle waiting to fill with other tasks
during a genuine number-crunching operation.

**Minimal example**

```python
import asyncio

async def fetch_price(ticker: str) -> float:
    print(f"Requesting price for {ticker}...")
    await asyncio.sleep(1)     # simulates a 1-second network call
    print(f"Got price for {ticker}")
    return 100.0

async def main():
    # Without concurrency, fetching 3 tickers sequentially takes ~3 seconds
    # await fetch_price("A"); await fetch_price("B"); await fetch_price("C")

    # With concurrency, all 3 "network calls" overlap — takes ~1 second total
    results = await asyncio.gather(
        fetch_price("RELIANCE.NS"),
        fetch_price("TCS.NS"),
        fetch_price("AAPL"),
    )
    print(results)

asyncio.run(main())
```

The key insight: while `fetch_price("RELIANCE.NS")` is "waiting" inside its
`await asyncio.sleep(1)`, the event loop is free to start
`fetch_price("TCS.NS")` and `fetch_price("AAPL")` as well — all three
"waits" overlap in real time instead of stacking up sequentially.

**Why a web API framework like FastAPI cares about this so much**

A production API server needs to handle many requests arriving at roughly
the same time. If handling one request involves waiting on I/O (reading a
file, calling a model registry, waiting on a database), a synchronous server
would leave the entire process stuck waiting for that one slow request
before it can even begin looking at the next one. An `async def` endpoint
lets the server start working on a second incoming request during the
"dead time" while the first request is still waiting on I/O — dramatically
improving how many concurrent users a single process can serve.

**The critical rule: don't put CPU-heavy work inside an `async def` endpoint**

Because `async`/`await` concurrency happens on a *single thread*, if you put
a genuinely CPU-intensive task (like training a neural network) directly
inside an `async def` function, it will **block the entire event loop** —
no other request can be handled at all until that CPU-heavy task finishes,
which defeats the entire purpose. The correct pattern is to run CPU-bound
work in a separate thread (or process) so the main event loop stays free to
keep handling other lightweight, I/O-bound requests.

**Where this becomes relevant in this project (previewed — full detail comes
in the Backend & Serving tier)**

- `serving/main.py`'s `/predict` endpoint is declared with `async def`,
  because loading model weights and reading feature files involves I/O.
- Genuinely CPU-heavy work — actually *training* a PPO/SAC agent, triggered
  via `/retrain/train` in `serving/retrain_router.py` — is explicitly kept
  **synchronous**, guarded by a `threading.Lock`-based `_training_active`
  flag, precisely to avoid the trap described above: a full training run is
  CPU-bound, so it should not silently block the event loop that's also
  trying to serve `/health` and `/predict` requests to other users at the
  same time.

---

## TIER 2 — Deep Learning (LSTM Return Forecaster)

### 1. Gradient Descent and Backpropagation

**What problem this solves**

A neural network starts with random numbers inside it (called weights). With
random weights, its predictions are just as random. Training means slowly
changing those weights so the predictions get better. Gradient descent is
the method used to change the weights in the right direction, step by step.

**How it works, in plain steps**

1. Feed some input data into the network. Get a prediction out.
2. Compare the prediction to the real answer. The difference is called the
   **loss** (or error). A big difference means a big loss.
3. Figure out, for every single weight in the network, "if I nudge this
   weight up a tiny bit, does the loss go up or down, and by how much?"
   This is called the **gradient**. A gradient is just a number that tells
   you the direction and size of the nudge that would make the loss worse.
4. Move every weight a small step in the *opposite* direction of its
   gradient (because we want the loss to go down, not up).
5. Repeat this thousands of times, using many different examples, until the
   loss gets small and stops improving much.

**Backpropagation — how the gradient is actually calculated**

A neural network is made of layers stacked on top of each other. The loss is
only calculated at the very end, after the last layer. But we need to know
the gradient for weights in *every* layer, including the very first one.

Backpropagation solves this using a rule from calculus called the **chain
rule**. It works backward from the loss: first it calculates how much the
last layer's weights affected the loss, then it uses that result to
calculate how much the second-to-last layer's weights affected the loss, and
so on, working backward layer by layer until it reaches the first layer.
This is why it's called "back"-propagation — the error signal flows
backward through the network, layer by layer, the opposite direction from
how the data flowed forward.

**Learning rate**

The learning rate controls how big each step is. Too big a step, and the
weights jump around wildly and never settle down (like overshooting a
target). Too small a step, and training takes forever because each step
barely moves anything. Picking a good learning rate is one of the most
important choices when training a network.

**Batch size**

Instead of updating the weights after looking at just one example, or
waiting until it has seen the entire dataset, a network is usually updated
after looking at a small group of examples at once — called a batch. A
batch size of 32 means the network looks at 32 examples, averages the
gradient across all of them, and then takes one step. This makes training
faster and the gradient estimate less noisy than using just one example.

**Gradient clipping — why we cap the gradient**

Sometimes the gradient calculated for a weight is a huge number — much
bigger than it should sensibly be. This can happen especially in networks
like LSTMs that process long sequences, where the chain rule multiplies many
numbers together and the result can explode. If you let a huge gradient push
the weights, training can become unstable and the model can suddenly get
much worse instead of better. Gradient clipping simply puts a ceiling on how
big a gradient step is allowed to be — if the calculated step is bigger than
the ceiling, it gets scaled down to the ceiling value before being applied.

**Where it's used in this project**

- `models/forecasting/train_forecaster.py::train_one_epoch()` runs exactly
  the steps above in a loop: `loss.backward()` triggers backpropagation and
  calculates all the gradients, `nn.utils.clip_grad_norm_(model.parameters(),
  max_norm=1.0)` is the gradient clipping step (capped at 1.0), and
  `optimizer.step()` is the actual weight update.
- The learning rate is set to `1e-3` in `DEFAULT_HP`, and the code also uses
  a **learning rate scheduler**
  (`torch.optim.lr_scheduler.ReduceLROnPlateau`) that automatically shrinks
  the learning rate if the validation loss stops improving — taking smaller,
  more careful steps once the model is close to a good answer.
- Batch size is set to `32` in the same hyperparameter dictionary.

---

### 2. LSTM — Long Short-Term Memory Networks

**What problem it solves**

A basic neural network has no memory of what came before — it only looks at
whatever input you give it right now. But price data is a sequence: today's
return depends partly on the pattern of returns over the past several
weeks, not just today's number alone. We need a network that can look at a
whole sequence of past days and remember useful patterns from earlier in
that sequence when making a prediction about the next day.

**What an LSTM is**

LSTM stands for Long Short-Term Memory. It is a type of network built
specifically to handle sequences (like a series of daily prices) one step
at a time, while carrying forward a kind of internal memory from one step
to the next.

**The three gates, explained simply**

An LSTM has a running "memory" that gets updated at every step in the
sequence. Three small internal switches, called **gates**, control what
happens to this memory at each step:

- **Forget gate** — decides what part of the old memory to throw away. For
  example, if the market regime just changed sharply, it might be useful to
  forget old patterns that no longer apply.
- **Input gate** — decides what new information from the current step
  should be added into the memory.
- **Output gate** — decides what part of the memory should actually be used
  to produce the output (the prediction) at this step.

Each gate is just a small learned function that outputs a number between 0
and 1 for each piece of information — closer to 0 means "block this," closer
to 1 means "let this through." The network learns, through training, what
values these gates should produce in different situations.

**Why this design helps with long sequences**

Without gates, a simple sequence network tends to either forget everything
useful very quickly, or the gradient calculated during backpropagation
shrinks to almost zero as it travels backward through many time steps
(a known problem called the vanishing gradient), meaning the network can't
learn from anything that happened far in the past. The gates give the LSTM
a controlled way to preserve important information over many steps, and to
also let the gradient flow backward through time without shrinking away to
nothing, which is why LSTMs work much better than simpler sequence networks
for problems where useful patterns can be spread out over many time steps.

**In this project — what pattern is the LSTM looking for**

The LSTM is given the last 30 days of features (returns, momentum,
volatility, RSI, and so on) for one stock, and is trained to predict what
tomorrow's return will be. The idea is that the model can learn patterns
such as **momentum** (a stock that's been going up tends to keep going up
for a while) and **mean-reversion** (a stock that's moved too far too fast
tends to pull back), by looking at how the features have been changing
across the 30-day window, not just at today's single snapshot.

**Where it's used in this project**

- `models/forecasting/lstm_model.py::ReturnLSTM` defines the network: a
  stacked LSTM (`num_layers=2`, `hidden_size=64`) followed by a small
  fully-connected head that turns the LSTM's final output into one number —
  the predicted next-day return.
- The input to the LSTM has shape `(batch_size, seq_len=30, input_size=10)`
  — 30 days of history, 10 features per day, for however many examples are
  in a batch. The model only reads the **very last** hidden state produced
  by the LSTM (`h_n[-1]`) to make its prediction, since that final state is
  meant to summarise everything useful the network picked up from the whole
  30-day sequence.
- `models/forecasting/dataset.py::ReturnSequenceDataset` is what builds
  these 30-day sliding windows out of the raw feature table before they're
  fed to the LSTM.

---

### 3. Dropout and Regularisation

**What problem this solves**

A model with a lot of learnable weights can sometimes do something bad
called **overfitting**: instead of learning the real, general pattern in the
data, it starts memorising quirks and noise specific to the training
examples it happened to see. A model like this looks great on the training
data (because it basically memorised it) but performs badly on new data it
hasn't seen before — which is exactly the situation you care about in
finance, since financial data is especially noisy and any given historical
sample only has a limited number of independent days.

**Regularisation, in general**

Regularisation is a general name for any technique that discourages a model
from overfitting — nudging it toward learning simpler, more general
patterns instead of memorising noise.

**Dropout, specifically**

Dropout is one common regularisation technique. During each training step,
a random fraction of the neurons in a layer are temporarily switched off
(set to zero) and ignored, as if they didn't exist for that one step. A
different random set of neurons gets dropped on the next step. This forces
the network to not rely too heavily on any single neuron or narrow
combination of neurons, because it never knows in advance which ones will
be missing. The result is a network that spreads its learned knowledge more
broadly and generalises better to new data.

Dropout is only active during training. When the model is actually being
used to make a real prediction (evaluation or live use), all neurons are
switched back on and used normally.

**Simple analogy**

Imagine a sports team where, at every practice, a random handful of players
are told to sit out. The remaining players are forced to learn how to play
well without relying on any one specific teammate always being there. By
game day, the whole team is more flexible and less dependent on any single
player, because everyone learned to cover for whoever might be missing.

**Where it's used in this project**

- `models/forecasting/lstm_model.py::ReturnLSTM` applies dropout in two
  places: `dropout=0.2` between the stacked LSTM layers themselves, and
  another `nn.Dropout(dropout)` inside the fully-connected prediction head.
  A dropout value of `0.2` means roughly 20% of neurons are randomly
  switched off at each training step.
- The comment in the code explicitly notes this is "critical for financial
  data which is noisy and has limited samples" — directly connecting the
  concept back to the specific risk described above.

---

### 4. Early Stopping

**What problem this solves**

If you keep training a model for more and more rounds (called **epochs** —
one epoch means the model has looked at every single training example once),
its performance on the *training* data will basically always keep slowly
improving. But at some point, its performance on data it hasn't seen (the
**validation** set) will stop improving and can even start getting *worse*
— this is overfitting happening in real time. Early stopping is a simple
way to catch this moment and stop training before the model overfits.

**How it works**

After every epoch, check the model's loss on the validation set (not the
training set). Keep a running record of the *best* validation loss seen so
far, and save a copy of the model's weights at that best point.

If the validation loss doesn't improve for a certain number of epochs in a
row — this number is called the **patience** — stop training entirely, and
go back to using the saved weights from the epoch that actually had the
best validation performance (not necessarily the very last epoch trained).

**Worked example**

Say patience is set to 7. If validation loss improves at epoch 12, then gets
slightly worse for epochs 13 through 19 (that's 7 epochs in a row with no
improvement), training stops after epoch 19, and the model reverts to the
weights it had saved back at epoch 12, since that was the actual best point.

**Why this is better than just guessing a fixed number of training epochs**

Picking a fixed number of epochs in advance (say, "always train for exactly
50 epochs") is risky — you might stop too early, before the model has
learned enough, or you might keep training long after the model has already
started overfitting, wasting time and ending up with a worse model than you
would have had if you'd stopped sooner. Early stopping automatically finds
close to the right stopping point for you, based on what's actually
happening during training, rather than a number chosen ahead of time.

**Where it's used in this project**

- `models/forecasting/train_forecaster.py::train_ticker()` implements this
  directly: it tracks `best_val_loss` and `patience_count`, and uses
  `hp["patience"] = 7` from the default hyperparameters. Every time
  validation loss improves, it saves a copy of the model's weights
  (`best_state = {k: v.clone() for k, v in model.state_dict().items()}`).
  Once `patience_count >= hp["patience"]`, the loop breaks early, and the
  saved best weights are reloaded (`model.load_state_dict(best_state)`)
  before the model is evaluated on the test set.

---

### 5. Directional Accuracy as an Evaluation Metric

**What problem this solves**

For a return-forecasting model, the most obvious way to measure error is
something like RMSE (root mean squared error) — how far off, on average,
were the predicted numbers from the actual numbers. But for a trading
strategy, getting the *exact* number right matters far less than getting
the *direction* right. If a stock actually goes up 0.5% and the model
predicts up 2%, the exact number was quite wrong, but the model still gave
you correctly useful information: "buy, this is going up."

**What directional accuracy measures**

Directional accuracy is simply: out of all the predictions made, what
fraction had the same sign (positive or negative) as the actual outcome? A
directional accuracy of 55% means the model correctly guessed "up" or
"down" 55% of the time.

**Why 50% is the baseline to beat, not 0%**

Since a return can only really go up or down, a model that predicts
completely at random (like flipping a coin) would still get the direction
right about 50% of the time purely by chance. So directional accuracy needs
to be compared against this 50% baseline, not against 0%. A model with 52%
directional accuracy is not necessarily "bad" — in finance, even a small,
consistent edge above 50%, applied repeatedly over many trades, can turn
into meaningful real returns. But a model needs to be judged relative to
that 50% baseline, not on an absolute scale.

**Why low error doesn't automatically mean useful predictions**

It's possible for a model to get a very low RMSE while still being useless
for trading. For example, if a stock's daily returns are almost always tiny
(close to zero, with the occasional larger move), a "lazy" model could just
predict a number very close to zero every single day. This would give it a
low RMSE (since it's rarely far off in absolute terms), but its directional
accuracy could be barely above 50%, or even below it, because a
near-zero-always prediction gives no real signal about whether tomorrow will
be up or down. This is exactly why the project checks directional accuracy
as a *separate*, additional metric, rather than relying on RMSE alone.

**Where it's used in this project**

- `models/forecasting/train_forecaster.py::directional_accuracy()` computes
  it with `np.mean(np.sign(y_true) == np.sign(y_pred))` — comparing whether
  the sign of the prediction matches the sign of the actual value, for every
  example, then averaging.
- It's tracked on both the validation and test sets during training, logged
  to MLflow, and used in `models/forecasting/evaluate.py::compare_models()`
  as the primary way of comparing the LSTM against the ARIMA baseline model
  — the comparison script literally counts how many tickers the LSTM "wins"
  on directional accuracy versus ARIMA.

---

## TIER 3 — Reinforcement Learning (Core of the Project)

### 1. Agent, Environment, State, Action, Reward Loop

**The basic idea**

Reinforcement learning (RL) is a way of training a decision-maker by letting
it try things, see what happens, and learn from the results — similar to how
a person might learn to play a new game by playing it repeatedly, rather
than being told the exact rules for winning in advance.

**The five core pieces**

- **Agent** — the decision-maker being trained. It's the thing that
  observes the current situation and picks an action.
- **Environment** — the world the agent interacts with. It reacts to the
  agent's actions and produces a new situation and a score.
- **State** — a snapshot describing the current situation the agent can see.
- **Action** — the choice the agent makes, given the current state.
- **Reward** — a number the environment hands back after each action,
  telling the agent how good or bad that action turned out to be.

**The loop, step by step**

1. The environment shows the agent the current state.
2. The agent picks an action based on that state.
3. The environment applies the action, moves to a new state, and returns a
   reward number.
4. The agent uses this reward to slowly adjust how it picks actions in the
   future, so that, over time, it favours actions that tend to lead to
   higher rewards.
5. Repeat, over and over, often for hundreds of thousands of steps.

**Simple analogy**

Think of training a dog. The dog is the agent. The room and the objects in
it are the environment. Whatever the dog currently sees and smells is the
state. Sitting, barking, or fetching are actions. A treat (or no treat) is
the reward. Over many repeated attempts, the dog learns which actions, in
which situations, tend to get it the most treats — without ever being
explicitly told the "rules."

**How this maps onto this specific project**

- **Agent** = the PPO or SAC trading policy being trained.
- **Environment** = `PortfolioEnv`, a simulation of the market built from
  real historical price data.
- **State** = the recent feature values (momentum, volatility, RSI, and so
  on) for every asset, plus the portfolio's current weights.
- **Action** = a new set of portfolio weights — how much of the portfolio's
  money to put into each asset.
- **Reward** = the portfolio's return for that day, minus a penalty for
  trading costs.

**Where it's used in this project**

- `models/rl_agent/portfolio_env.py::PortfolioEnv` is the environment. Its
  `step()` method is exactly this loop's "apply the action, return the new
  state and reward" logic, called once per simulated trading day.
- `models/rl_agent/train_agent.py::train()` is where the agent (PPO or SAC,
  from the Stable-Baselines3 library) actually repeats this loop hundreds of
  thousands of times to learn.

---

### 2. Policy Gradient Methods — PPO

**What a "policy" is**

A policy is just the agent's strategy — a function that looks at the
current state and outputs an action. At the start of training, the policy
is basically random. Training slowly reshapes the policy so it tends to
output actions that lead to higher rewards.

**What "policy gradient" methods do**

A policy gradient method directly adjusts the policy's own internal weights
(similar to the weights inside a neural network) in the direction that
should increase expected future reward — using a version of the same
gradient descent idea from Tier 2, but now the "loss" being minimised is
built from the rewards the agent has been collecting, instead of a
prediction error against a known correct answer.

**What PPO is and why it was designed the way it was**

PPO stands for Proximal Policy Optimisation. The core problem PPO solves:
if you let the policy change *too much* in a single update (based on a
single batch of recent experience), the new policy can become much worse
than the old one, sometimes catastrophically, and the whole training process
can become unstable and never recover.

PPO fixes this by **clipping** how far the policy is allowed to change in a
single update step. It compares the new policy's behaviour to the old
policy's behaviour, and if an update would push the new policy too far away
from the old one, that update gets capped — it's only allowed to move so
far, no matter how strongly the raw calculation says it should. This keeps
training much more stable, at the cost of taking slightly smaller, more
careful steps.

**On-policy — what this means for PPO**

PPO is called an **on-policy** method. This means it can only learn from
experience that was collected using something very close to its *current*
policy. Once the policy is updated, the older experience gathered under the
previous policy version is no longer used, and fresh experience has to be
collected with the new version before the next update. This makes PPO
somewhat data-hungry (it can't reuse old experience for very long) but
generally quite stable and simple to tune, which is why it's a very common
default choice for problems like this one.

**Why continuous action spaces suit PPO well**

Some RL problems have a small, fixed list of possible actions (like "move
left" or "move right" in a simple game). This project's action, portfolio
weights, is instead **continuous** — an infinite range of possible values
for each asset's allocation. PPO handles continuous action spaces
comfortably, which is part of why it's used as the primary algorithm here.

**Where it's used in this project**

- `models/rl_agent/train_agent.py` uses Stable-Baselines3's built-in `PPO`
  class, trained for `100,000` environment steps (`TIMESTEPS["ppo"]`), with
  hyperparameters including `clip_range: 0.2` — this is exactly the clipping
  amount described above, limiting how far a single policy update is allowed
  to move.
- PPO is described in the project as the **primary** RL algorithm, with SAC
  used as a secondary "challenger" candidate.

---

### 3. SAC — Soft Actor-Critic

**What makes SAC different from PPO**

SAC is also a method for training a policy, but it works quite differently
under the hood, and it's useful to understand the contrast with PPO.

**Off-policy — and why this matters**

SAC is an **off-policy** method. Unlike PPO, SAC can learn from experience
collected a long time ago, using an *older* version of the policy, not just
its current one. This is possible because of something called a **replay
buffer**.

**Replay buffer, explained simply**

A replay buffer is just a large storage of past experience — states,
actions, and rewards the agent has seen before, going back potentially a
long way. Instead of only learning from the most recent batch of
experience (like PPO does), SAC repeatedly pulls random samples from this
whole stored history to keep learning from. This tends to make SAC more
**sample-efficient** — it can squeeze more learning progress out of the same
amount of collected experience, since it reuses old data instead of
throwing it away after one use.

**Entropy — encouraging exploration**

SAC has a specific extra goal built into its training: alongside trying to
maximise reward, it also tries to keep the policy's actions somewhat
**random/spread-out** rather than becoming too narrow and repetitive too
quickly. This "spread-outness" is measured using a concept called
**entropy** — high entropy means the policy's chosen actions vary a lot;
low entropy means the policy has become very confident and repetitive,
almost always picking the same kind of action. By explicitly rewarding some
amount of entropy, SAC encourages the agent to keep exploring different
possible actions during training, rather than locking onto one narrow
strategy too early and missing a potentially better one.

**Simple analogy for exploration vs exploitation**

Imagine trying to find the best restaurant in a new city. If you go back to
the very first decent restaurant you find every single time (pure
exploitation), you might miss an even better one nearby that you never
tried. If you always try a totally random new place every time (pure
exploration), you never get to enjoy the good ones you've already found.
Entropy in SAC is a way of nudging the agent to keep trying a healthy mix of
different things during training, instead of settling too early.

**Where it's used in this project**

- `models/rl_agent/train_agent.py` trains SAC (Stable-Baselines3's `SAC`
  class) for `150,000` environment steps — more steps than PPO's 100,000,
  reflecting that SAC is generally used here as the secondary "challenger"
  algorithm being compared against the primary PPO agent.
- The hyperparameters include `buffer_size: 100_000` (the size of the replay
  buffer described above) and `ent_coef: "auto"` (the entropy coefficient,
  set to be tuned automatically during training rather than fixed by hand).

---

### 4. Action Space — Simplex Constraint and Softmax Projection

**The problem: portfolio weights have rules they must follow**

A valid set of portfolio weights must satisfy two rules: every individual
weight must be zero or positive (assuming no short-selling), and all the
weights together must add up to exactly 1 (100% of the money is allocated
somewhere). A set of numbers that follows both these rules is said to lie on
something called the **simplex** — just a mathematical name for "all
possible valid weight combinations."

The tricky part: if you let a neural network output raw numbers directly as
the portfolio weights, there's nothing forcing those raw numbers to be
positive, or to add up to 1. The network could easily output something like
`[0.7, -0.3, 0.9]`, which isn't a valid portfolio at all.

**The solution used in this project: raw logits plus softmax**

Instead of asking the network to output valid weights directly, the network
is allowed to output any raw numbers it wants — called **logits** — ranging
anywhere from -1 to 1 in this project. These raw numbers are then passed
through a function called **softmax**, which converts *any* list of numbers
into a valid set of weights that are all positive and sum to exactly 1.

**How softmax works, step by step**

1. Take the raw numbers (logits).
2. Raise the mathematical constant `e` to the power of each number
   (exponentiate them). This step guarantees every result is positive, no
   matter what the original number was — even a negative logit turns into a
   small positive number after this step.
3. Divide each of these exponentiated numbers by the sum of all of them.
   This step guarantees everything now adds up to exactly 1.

**Worked example**

Say the raw logits for 3 assets are `[2.0, 1.0, 0.1]`.

- Exponentiate: `[7.39, 2.72, 1.11]`
- Sum: `7.39 + 2.72 + 1.11 = 11.22`
- Divide each by the sum: `[0.66, 0.24, 0.10]`

These three final numbers are all positive and add up to 1 — a valid
portfolio. Notice that a bigger raw logit turns into a bigger final weight —
softmax preserves the *order* of the original numbers while reshaping them
into valid weights.

**Why this is a cleaner approach than directly constraining the network's
output**

You could try to force the network's raw output to already sum to 1 (for
example, by adding extra rules or penalty terms during training). But this
tends to create messy training problems near the "edges" of what's allowed
— for example, what happens if the network tries to output a weight
slightly less than 0? Softmax avoids all of this: the network is free to
output *any* numbers it likes without any restriction at all, and the
softmax step afterward always guarantees a clean, valid portfolio no matter
what came in. This keeps training smoother and avoids edge-case problems.

**Temperature**

Softmax has an extra setting called **temperature**, which controls how
"sharp" or "spread out" the final weights are. A very low temperature makes
softmax behave almost like "give everything to the single largest number, and
nothing to the rest." A high temperature makes softmax spread the weights
out much more evenly, even if the input numbers were quite different from
each other. This project uses a temperature of `1.0` (the default,
untouched setting).

**Where it's used in this project**

- `models/rl_agent/portfolio_env.py::softmax()` is exactly the three-step
  calculation above (with an extra small trick — subtracting the maximum
  value before exponentiating — purely to avoid numbers getting too large
  for the computer to handle; this doesn't change the final answer at all).
- The action space itself is declared as
  `spaces.Box(low=-1.0, high=1.0, shape=(n_assets,))` in `PortfolioEnv`,
  matching the "any raw logits between -1 and 1" description above — the
  environment's `step()` method calls `softmax(action)` immediately after
  receiving the agent's raw action, before doing anything else with it.

---

### 5. Episode Design — Length and Start Date Randomisation

**What an "episode" is**

An episode is one complete run of the environment, from a starting point to
an ending point, made up of many individual steps. In this project, one
episode represents the agent trading for a stretch of simulated time before
the environment resets back to the beginning of a brand new episode.

**Episode length: 252 steps**

Each episode lasts 252 steps, matching the number of trading days in
roughly one calendar year. This gives the agent a long enough stretch within
a single episode to experience meaningful ups and downs in the market,
rather than only ever seeing very short, disconnected snippets of price
action.

**Why the start date is picked randomly on every reset**

Every time a new episode begins, the environment picks a **random** starting
date somewhere within the available training period, rather than always
starting from the exact same fixed date. This is a deliberate design choice,
and it matters a lot.

**The problem this solves: memorisation of one specific path**

If the agent always started at the exact same date every single episode, it
could start to simply memorise the specific sequence of prices that follow
that one starting date — essentially "learning the answer" for that one
particular slice of history, rather than learning a general trading
strategy that works across many different kinds of market conditions.

By picking a different random starting point every episode, the agent gets
exposed, across its many training episodes, to a much wider variety of
situations pulled from different parts of history — some episodes will
happen to start right before a market crash, some during a calm stretch,
some during a strong uptrend, and so on. This pushes the agent toward
learning a strategy that generalises across different kinds of market
behaviour, instead of overfitting to one specific historical path.

**Simple analogy**

Imagine training for a driving test by always practising on the exact same
one-mile stretch of road. You might become excellent at that specific
stretch without actually becoming a good driver in general. Practising on
many different, randomly chosen roads instead forces you to learn skills
that generalise to roads you've never driven before.

**Where it's used in this project**

- `models/rl_agent/portfolio_env.py` sets `EPISODE_LENGTH = 252` at the top
  of the file, and inside `reset()`:
  `self._start_idx = int(self.np_random.integers(low=0,
  high=max(1, self.train_end_idx)))` — this is the random starting point
  being chosen, using a random number generator, every single time an
  episode resets.
- The random start is deliberately limited to only pick starting points that
  are *within the training window* (`self.train_end_idx`), so the agent
  never accidentally trains on data from the validation or test period —
  connecting directly back to the data leakage discussion in Tier 0.

---

### 6. Reward Function Design — Why It's Hard

**Why this is considered one of the hardest parts of building an RL system**

The reward function is the *only* signal the agent gets about what "good"
behaviour looks like. Everything the agent eventually learns to do is
shaped entirely by what gets rewarded and what doesn't. If the reward
function is badly designed, the agent will still learn something — but it
might learn to game the reward in a way that technically scores well while
doing something completely unhelpful or even harmful in the real world.
Getting the reward function right is often harder, and more important, than
picking which specific algorithm (PPO vs SAC) to use.

**Problem 1 — using Sharpe ratio directly as a reward causes instability**

As discussed already in Tier 1, a plain Sharpe ratio penalises *all*
volatility equally, including good, upward volatility. If used directly as
the reward, this actively discourages the agent from taking on any large
positive-return opportunities, which is exactly the opposite of what you
want a trading agent to learn.

**Problem 2 — Sortino ratio calculated over a very short window is noisy**

Sortino fixes the "penalising good volatility" problem, but there's a
separate issue: if you calculate a ratio like Sortino using only a small
handful of recent returns (say, just the last few days), the resulting
number can bounce around wildly just from random noise, rather than
reflecting anything meaningful about the quality of the agent's decisions.
A reward signal that swings around due to noise, rather than due to the
agent's actual choices, makes training much harder and less stable.

**Problem 3 — ignoring transaction costs teaches unrealistic behaviour**

If the reward function only cares about raw return and completely ignores
the cost of trading, the agent has no reason at all to avoid excessive,
unnecessary trading. It might learn to reallocate the whole portfolio every
single day chasing tiny, noisy signals — behaviour that would rack up real
trading costs and destroy actual profitability, even though it looks
harmless inside a simulation that doesn't model those costs.

**The solution used in this project: a blended, dense, per-step signal**

Rather than one single perfect formula, this project's actual reward
(`step_reward()` in `reward.py`) combines a few pieces, kept simple enough
to give the agent useful feedback at every single step:

```
reward = log_return_this_step  −  (transaction_cost_rate × turnover_this_step)
```

This gives the agent an immediate, dense reward signal every single day
(rather than only at the very end of a 252-day episode), while still baking
in the cost-of-trading penalty from Tier 1 directly into what the agent is
optimising for.

Separately, at a higher level — when *judging* how good a finished episode
or a finished training run was (not the moment-to-moment training signal
itself) — the project uses the full episode-level Sortino ratio
(`sortino_reward()`, and `evaluate_agent.py::compute_metrics()`), since
Sortino is a better *summary* measure of quality once you have a full
episode's worth of returns to calculate it from, even though it's too noisy
to use as the tiny, per-step training signal.

**Reward normalisation**

The project's documentation also mentions reward normalisation as part of
what stabilises training — meaning the raw reward numbers are kept on a
sensible, consistent scale (not wildly large or wildly small) before being
used to update the policy, since extremely large or inconsistent reward
values can make gradient-based training unstable, the same way an
unreasonably large learning rate can (see Tier 2).

**Where it's used in this project**

- `models/rl_agent/reward.py::step_reward()` is the exact per-step formula
  shown above, called on every single step inside `PortfolioEnv.step()`.
- `models/rl_agent/reward.py::sortino_reward()` is the episode-level version
  used for evaluation, not the live training signal.

---

### 7. OpenAI Gym / Gymnasium Interface

**What this is and why it exists**

Gym (and its newer, actively maintained version, Gymnasium) is a shared,
standard way of building RL environments, so that any RL algorithm library
(like Stable-Baselines3, used in this project) can work with *any*
environment that follows the same standard set of rules, without needing
custom code written for each new environment. It's essentially an agreed
"shape" that every environment must follow, so algorithms and environments
can be mixed and matched freely.

**The required pieces of the interface**

- **`reset()`** — starts a brand new episode. Returns the starting
  observation (state) and a small dictionary of extra info. Must be called
  before an episode can begin.
- **`step(action)`** — takes one action from the agent and advances the
  environment by one step. Returns five things: the new observation, the
  reward for that step, whether the episode has **terminated** (ended
  naturally, for example the agent "died" in a game), whether the episode
  has been **truncated** (cut off for an external reason, like hitting a
  maximum step limit, even though nothing specifically "ended" it), and an
  info dictionary with any extra details.
- **`observation_space`** and **`action_space`** — these describe, in
  advance, the *shape* and *valid range* of every observation and every
  action the environment will ever produce or accept. This lets an
  algorithm library automatically build a neural network with the correctly
  sized input and output layers, without a human needing to hard-code those
  sizes by hand.

**Terminated vs truncated — why this distinction exists**

This is a subtle but useful distinction. "Terminated" means the episode
reached a genuine natural end according to the rules of the task itself
(for example, a game character running out of lives). "Truncated" means the
episode was stopped for a reason external to the task's own rules — most
commonly, simply because a fixed maximum number of steps was reached. This
project's episodes always end by truncation (hitting the 252-step limit),
never by "termination," since there's no natural in-task ending condition
for portfolio trading — the market simply keeps going.

**Why an automatic checker tool is useful**

Because so many different algorithms rely on an environment correctly
following this interface, Stable-Baselines3 provides a built-in checker
tool that automatically tests a custom environment against the full set of
interface rules — correct shapes, correct return types, no NaN values
appearing anywhere, and so on — catching mistakes early, before spending
hours or days on actual training with a broken environment.

**Where it's used in this project**

- `models/rl_agent/portfolio_env.py::PortfolioEnv` is built as a subclass of
  `gym.Env` and implements exactly this interface: `reset()`, `step()`,
  `render()`, and `close()`, along with `observation_space` and
  `action_space` declared using Gymnasium's `spaces.Box` class.
- `models/rl_agent/env_checker.py` runs Stable-Baselines3's built-in
  `check_env()` function against `PortfolioEnv`, plus an extra manual
  5-episode sanity test that checks things like "do the portfolio weights
  always sum to exactly 1" and "is the reward ever `NaN`" — confirming the
  environment is correctly built *before* spending time on real training in
  `train_agent.py`.

---

## TIER 4 — MLOps (What Separates Juniors from Seniors in Interviews)

### 1. MLflow — Experiments, Runs, Artifacts, Model Registry

**The problem MLflow solves**

When you're training models, you end up trying many different versions —
different hyperparameters, different tickers, different algorithms (PPO vs
SAC). Without a system to track this, you end up with a messy folder of
files named things like `model_v2_final_ACTUALLY_final.zip`, and no real
record of which settings produced which result, or which model is currently
the "best" one. MLflow is a tool that automatically keeps an organised
record of every training attempt, so you never lose track of what you did
and why.

**The four building blocks**

- **Experiment** — a named folder-like grouping for a specific kind of
  training task. For example, all LSTM training attempts go under one
  experiment called `lstm_forecaster`; all RL training attempts go under a
  different experiment called `rl_agent_training`. This keeps unrelated
  training work separated.

- **Run** — one single attempt at training, inside an experiment. Every
  time you train, that's a new run. A run records three things:
  - **Parameters** — the settings you used going in (learning rate, batch
    size, which ticker, which algorithm). These never change once the run
    starts.
  - **Metrics** — the numbers that come out (validation loss, Sharpe ratio,
    directional accuracy). These can be logged multiple times as training
    progresses, so you can see how a metric changed over the training
    process, not just its final value.
  - **Artifacts** — actual files produced by the run, most importantly the
    trained model file itself, but also things like plots or logs.

- **Model Registry** — a separate, higher-level catalogue that sits on top
  of individual runs. Instead of remembering "the good model is buried
  inside run `a1b2c3`," you register a model into the registry under a
  simple name (like `PortfolioAgent`), and the registry keeps track of every
  version of that model that's ever been registered, in order.

- **Alias** (like `champion`) — a movable label you can attach to any one
  specific version inside the registry. Instead of your serving code needing
  to know "use version 17 specifically," it can just ask for "whichever
  version currently has the `champion` label." When a better model comes
  along, you simply move the `champion` label to the new version — nothing
  in the serving code needs to change at all.

**Worked example**

Imagine training 5 different PPO agents with slightly different settings.
Each one becomes a separate **run** inside the `rl_agent_training`
**experiment**, each with its own logged parameters (like `learning_rate:
3e-4`) and metrics (like `val_sharpe: 1.42`). After comparing all 5 runs,
you pick the best one and register it as version 1 of the `PortfolioAgent`
model, then attach the `champion` alias to that version. Later, a 6th run
does even better — you register it as version 2, and simply move the
`champion` alias from version 1 to version 2.

**Where it's used in this project**

- `models/rl_agent/train_agent.py::train()` logs everything: parameters via
  `mlflow.log_param()`, metrics like `val_sharpe` via `mlflow.log_metric()`,
  and the actual trained policy as an artifact via
  `mlflow.pytorch.log_model(model.policy, ..., registered_model_name=
  "PortfolioAgent")` — this single call both saves the model artifact and
  registers it into the Model Registry in one step.
- `serving/model_loader.py::ModelRegistry._load_champion()` reads the model
  back out using `client.get_model_version_by_alias(MODEL_NAME, "champion")`
  — asking for "whichever version currently holds the champion alias,"
  exactly as described above.

---

### 2. Champion-Challenger Model Promotion

**The core idea**

Champion-challenger is a simple, safe pattern for deciding when to actually
replace the model that's live in production. The current best model is
called the **champion**. Every newly trained model is called a
**challenger**. A challenger only gets promoted to become the new champion
if it's *meaningfully* better than the current champion — not just
marginally, randomly better.

**Why you shouldn't promote on any small improvement**

Training involves randomness — different random starting weights, different
random episodes sampled during RL training, and so on. If you retrain the
exact same model twice with the exact same settings, you'll likely get two
slightly different Sharpe ratios just from this randomness, even though
neither one is truly "better" in any meaningful sense. If you promoted a new
model every single time it scored even 0.001 higher than the current
champion, you'd be constantly swapping models based on pure noise — this is
called **model churn**, and it makes the system unstable and unpredictable
without any real benefit.

**The 5% improvement threshold used in this project**

To avoid promoting on noise, a challenger must beat the champion's
validation Sharpe ratio by at least 5% before it's promoted:

```
promote_if:  new_model_sharpe  >  champion_sharpe × 1.05
```

**Worked example**

If the current champion has a validation Sharpe of `1.20`, a challenger
needs a Sharpe higher than `1.20 × 1.05 = 1.26` to be promoted. A
challenger scoring `1.22` — technically higher, but likely just noise — does
**not** get promoted; the current champion stays in place. A challenger
scoring `1.35` clears the bar and does get promoted.

**Special case: no champion exists yet**

The very first time a model is trained, there's no champion to compare
against yet. In that case, the new model is automatically promoted
regardless of its score, simply because "some model" is needed before the
system can serve any predictions at all.

**Where it's used in this project**

- `models/rl_agent/train_agent.py::get_current_champion_sharpe()` fetches
  the current champion's stored Sharpe from MLflow, defaulting to `0.0` if
  no champion exists yet (which then triggers the "no champion yet, promote
  automatically" case).
- The actual comparison and promotion logic:
  ```python
  threshold = current_champion_sharpe * (1 + CHAMPION_IMPROVEMENT_THRESHOLD)
  if new_sharpe > threshold or current_champion_sharpe == 0.0:
      client.set_registered_model_alias(MODEL_NAME, "champion", latest.version)
  ```
  with `CHAMPION_IMPROVEMENT_THRESHOLD = 0.05` — exactly the 5% rule
  described above.
- The same logic is duplicated inside
  `serving/retrain_router.py::evaluate_champion()`, since that's the
  endpoint Airflow actually calls during the nightly retraining pipeline
  (details in the next section).

---

### 3. Model Drift Detection

**The core idea**

A model trained on past data is implicitly assuming that the future will
behave somewhat similarly to the past it learned from. But markets change —
new regimes emerge, volatility patterns shift, relationships between assets
change. When the real world starts behaving differently from what a model
learned, the model's performance quietly degrades over time. This is called
**drift**. Drift detection means continuously watching for signs that this
is happening, so the model can be refreshed before it becomes actively
harmful rather than just slightly stale.

**Why you can't just "check it once and trust it forever"**

A model that performed excellently when it was first deployed can silently
get worse and worse as market conditions evolve away from what it was
trained on, without anyone necessarily noticing right away — especially if
nobody is actively watching its live performance. Drift detection is what
turns "hope the model is still good" into "actually know, with a number,
whether the model is still good."

**How this project measures drift: rolling 30-day Sharpe ratio**

Rather than comparing every single day's prediction to some fixed
benchmark, the system keeps a running record of the model's most recent 30
days of live returns, and continuously recalculates the Sharpe ratio over
just that rolling 30-day window. If this recent, live Sharpe ratio starts
dropping meaningfully below what the model achieved during backtesting, that's
a signal the model's real-world performance has drifted away from what it
was trained to do.

**The threshold: 0.5**

If the rolling 30-day Sharpe drops below `0.5`, that's treated as a drift
alert, triggering an automatic retraining cycle. This threshold was chosen
to sit roughly one standard deviation below the model's backtested Sharpe
(around 1.4–1.5) — low enough to avoid false alarms from normal short-term
noise, but high enough to catch a genuine, sustained decline in performance
before it gets much worse.

**Where it's used in this project**

- `serving/metrics.py` keeps a `deque(maxlen=30)` (a fixed-size rolling
  buffer that automatically drops the oldest entry once it's full) of
  recent daily portfolio returns, updated by `record_portfolio_return()`,
  and recalculates the rolling Sharpe via `_update_rolling_sharpe()`,
  exposed as the Prometheus gauge `portfolio_rolling_sharpe_30d`.
- This gauge is exactly what Grafana watches to decide when to fire an
  alert (covered fully in the Prometheus section below), which then
  triggers `mlops/monitoring/alert_trigger.py`, which in turn triggers the
  Airflow retraining DAG.

---

### 4. Airflow as Orchestrator — Not a Training Environment

**What Airflow actually is**

Airflow is a tool for scheduling and coordinating multi-step workflows — it
runs a sequence of tasks in the right order, at the right time, and keeps
track of whether each step succeeded or failed. Think of Airflow as a very
reliable, very well-organised assistant who knows the checklist of steps
that need to happen every night, and calls the right people to actually do
each step, in the right order, retrying if something fails.

**The common mistake this project deliberately avoids**

A very common mistake in real MLOps setups is putting the actual heavy
machine-learning code (PyTorch, Stable-Baselines3, and so on) directly
inside Airflow itself. This creates several real problems:

- Airflow's own container/environment becomes enormous (installing PyTorch,
  Stable-Baselines3, cvxpy, and every other heavy ML library), even though
  Airflow's actual job — scheduling — doesn't need any of them.
- Every ML library upgrade risks breaking Airflow's own scheduler, since
  they'd all be tangled together in the same environment.
- If a long training job crashes Airflow's worker process, the entire
  scheduling system for *every other* workflow can go down with it, not
  just the one training job.

**This project's approach: Airflow only makes HTTP calls**

Instead of running any ML code itself, Airflow's DAG (the workflow
definition) does nothing but send HTTP requests to the FastAPI service,
which is the system that actually has all the heavy ML libraries installed
and does all the real work. Airflow's job is reduced to: "call this web
address, wait for a response, then call the next web address in order."

**Simple analogy**

Think of Airflow as a restaurant's kitchen manager who doesn't cook
anything personally — they just call out orders to the chefs (the FastAPI
service) at the right time, in the right sequence, and keep track of
whether each dish came out okay. The manager doesn't need to own a stove or
know how to cook; they just need to know *when* to tell someone else to
start cooking.

**The actual nightly sequence**

```
fetch_new_data → engineer_features → [train_ppo, train_sac run together] → evaluate_champion → notify_result
```

Notice `train_ppo` and `train_sac` run **in parallel** — since they're
independent of each other, there's no reason to make one wait for the
other to finish before starting.

**Where it's used in this project**

- `mlops/airflow/dags/retrain_dag.py` contains, deliberately, **zero
  machine-learning code**. Every task function
  (`task_fetch_data`, `task_train_ppo`, and so on) does nothing but call
  `call_training_service()`, which is just a wrapped `requests.post()` call
  to an endpoint like `http://api:8000/retrain/train`.
- The actual DAG structure — `fetch >> engineer >> [train_ppo, train_sac] >>
  evaluate >> notify` — is the parallel-training sequence described above,
  using Airflow's `>>` operator to declare "this must run after that."
- The real ML work lives entirely in `serving/retrain_router.py`, inside the
  FastAPI service, which does have all the heavy ML libraries installed,
  exactly the split described above.
- The schedule `"30 12 * * 1-5"` (a cron expression) means: run at 12:30 UTC
  (18:00 IST) on weekdays only (Monday through Friday, represented as
  `1-5`).

---

### 5. Prometheus Pull Model vs Push Model

**Two different ways a monitoring system can collect metrics**

- **Push model**: the application itself is responsible for actively
  sending its metrics out to a central monitoring server, whenever it
  decides to. The application has to know the monitoring server's address
  and remember to send updates.
- **Pull model**: the monitoring server instead reaches *out* to each
  application at regular intervals and asks, "what are your current metrics
  right now?" The application doesn't need to know anything about the
  monitoring server at all — it just needs to expose its current numbers at
  a known address, and let the monitoring server come and fetch them
  whenever it wants.

**Prometheus specifically uses the pull model**

Prometheus is a monitoring tool that works by periodically sending an HTTP
request to a `/metrics` address on each application it's watching, reading
back whatever numbers are currently there, and storing them with a
timestamp. The application's only job is to always have an up-to-date
`/metrics` page ready to be read — it never has to actively "send" anything
anywhere.

**Why the pull model is a meaningfully different design than logging**

This is worth being explicit about, since it's easy to blur together with
regular application logging. A log line is written once, at the moment
something happens, and then it just sits in a log file or log stream. A
Prometheus metric, by contrast, is a **live, current value** that always
reflects "right now" — every time Prometheus asks, it gets whatever the
value currently is at that instant, not a historical record of every past
event. Prometheus itself is the one responsible for repeatedly sampling
this live value over time and building up a history from those samples.

**The three kinds of metrics used in this project**

- **Counter** — a number that only ever goes up (or resets to zero on
  restart), never decreases. Good for counting things like "total number of
  requests handled so far."
- **Gauge** — a number that can go up or down freely, representing a
  current value at this moment. Good for something like "today's most
  recent portfolio return" or "current rolling Sharpe ratio."
- **Histogram** — groups measurements into buckets (ranges), so you can see
  the *distribution* of values, not just a single average. Good for request
  latency, where you often care about questions like "how many requests
  took longer than 200ms," not just the average latency across all
  requests.

**Where it's used in this project**

- `serving/metrics.py` defines exactly these three kinds:
  `REQUEST_COUNT` (a Counter, labelled by success/error status),
  `PORTFOLIO_RETURN` and `ROLLING_SHARPE` (both Gauges), and
  `REQUEST_LATENCY` (a Histogram, with buckets at
  `[10, 25, 50, 100, 200, 500, 1000]` milliseconds, letting you see the
  latency distribution and directly check the project's target of
  "p99 latency under 200ms").
- `serving/main.py::metrics()` exposes all of this at `GET /metrics`,
  formatted in the specific text format Prometheus expects
  (`generate_latest()` from the `prometheus_client` library).
- `mlops/monitoring/prometheus.yml` is the pull-side configuration: it tells
  Prometheus to reach out to `targets: ["api:8000"]` at `metrics_path:
  "/metrics"` every `scrape_interval: 15s` — Prometheus initiating the
  pull, exactly as described above, not the API pushing anything to
  Prometheus.

---

## TIER 5 — Backend and Serving

### 1. REST API Design — Endpoints, HTTP Methods, Status Codes

**What a REST API is, in plain terms**

An API (Application Programming Interface) is just a defined way for two
programs to talk to each other over the network — one program sends a
request, the other sends back a response. A REST API is a common style of
designing these APIs around simple web addresses (called **endpoints**) and
a small, standard set of actions.

**HTTP methods — the "verb" of a request**

Every request to a REST API includes a method that says what kind of action
is being requested:

- **GET** — asking for information, without changing anything on the
  server. Safe to call repeatedly; it won't cause side effects.
- **POST** — asking the server to *do* something, usually creating
  something new or triggering an action that changes state.

There are other methods (PUT, DELETE, and so on) used in many APIs, though
this project's API only needs GET and POST for its specific endpoints.

**Endpoints — the specific "address" being asked**

An endpoint is a specific path on the server, like `/predict` or `/health`.
Combined with a method, this fully describes one specific action: `GET
/health` means "check if the service is alive"; `POST /predict` means
"please compute and return a portfolio allocation."

**Status codes — a standard way to say what happened**

Every response includes a numeric status code that tells the caller, at a
glance, what happened:

- **200** — success, everything worked as expected.
- **401** — authentication failed (missing or wrong API key).
- **422** — the request itself was invalid (for example, missing a required
  field, or the wrong data type).
- **500** — something went wrong inside the server itself while handling an
  otherwise valid request.
- **503** — the service is temporarily not ready to handle this kind of
  request right now (for example, the model hasn't finished loading yet).

**Why using the right status code actually matters**

A calling program (or a human debugging a problem) can react automatically
based on the status code alone, without needing to read and parse a
free-text error message. A `401` immediately tells you "check your API
key." A `503` tells you "this isn't a real failure, just try again in a
moment." A `422` tells you "look at what you sent, something about it was
invalid." Picking the correct status code is part of making an API easy and
predictable to work with.

**Where it's used in this project**

- `serving/main.py` uses `GET /health` (always returns 200 if the process is
  running at all — a basic "is this thing even alive" check), `GET /ready`
  (returns 200 only if the model has finished loading, otherwise 503), and
  `POST /predict` (the actual prediction action, requiring authentication).
- Specific status codes are raised explicitly in code: `raise
  HTTPException(status_code=401, ...)` for a bad API key, `raise
  HTTPException(status_code=422, ...)` when `predict_weights()` raises a
  `ValueError` (for example, too few valid tickers were given), and `raise
  HTTPException(status_code=500, ...)` for any other unexpected internal
  error.

---

### 2. Pydantic — Data Validation at API Boundaries

**The problem this solves**

An API can be called by anyone, sending anything — a mistyped field name, a
number where text was expected, a completely missing required field, or
even something malicious. If invalid or malformed data is allowed to flow
straight through into the actual business logic (in this case, the machine
learning prediction code), it can cause confusing crashes deep inside the
system, or in the worst case, silently produce wrong results without any
error at all. Pydantic is a tool that checks incoming data *before* any real
logic runs, rejecting anything that doesn't match what's expected.

**How it works**

You define a "shape" for the data you expect — field names, their types,
and any extra rules — as a simple class. FastAPI automatically uses this
shape to check every incoming request: if the request matches the expected
shape, it gets converted into a normal Python object and passed to your
function. If it doesn't match, FastAPI automatically sends back a `422`
error, with a clear description of exactly what was wrong, and your actual
function code never even gets called.

**Worked example, matching this project**

The API expects a list of ticker strings, with at least 2 and at most 50
entries. If someone sends only 1 ticker, or sends a number instead of a
string, Pydantic catches this automatically and returns a clear `422` error
message like "list should have at least 2 items" — without any custom
error-handling code needing to be written by hand for this specific case.

**Why this matters specifically before ML code runs**

Machine learning code is often not written to gracefully handle
unexpected, malformed input — it usually just assumes the input is already
correct and well-formed, and can crash in confusing ways (or silently
produce garbage) if it isn't. By validating everything at the API boundary,
*before* any ML code ever sees the data, you guarantee that whatever
reaches the prediction logic is already guaranteed to be well-formed,
letting the ML code stay simple and not have to defensively check its own
inputs.

**Where it's used in this project**

- `serving/schemas.py::PredictRequest` defines exactly this: `tickers:
  list[str] = Field(..., min_length=2, max_length=50, ...)`, plus a custom
  `@field_validator` that strips whitespace and rejects empty ticker
  strings, converting every ticker to uppercase automatically along the
  way.
- `PredictResponse`, `HealthResponse`, and `ReadyResponse` define the exact
  shape of what the API promises to send *back*, too — Pydantic validates
  both the incoming request and the outgoing response, so the API's actual
  behaviour always matches its documented contract.

---

### 3. Liveness vs Readiness Probes

**Why you need two different "is it working?" checks, not just one**

It's tempting to think a service is simply either "up" or "down." In
practice, there's an important middle state: the server process itself has
started and is running, but it isn't fully ready to correctly handle real
requests yet — for example, because a large model file is still being
loaded into memory. Treating "process is running" and "ready to serve real
traffic" as the same thing causes real problems, which is why two separate
checks exist.

**Liveness probe — "is the process alive at all?"**

A liveness check answers a very basic, low-level question: is the server
process still running, and responding to *any* request at all? It should
almost always return success (yes, this project's `/health` always returns
200 as long as the process is up), and it exists mainly so that an
orchestration system (like Docker or Kubernetes) can detect a truly frozen
or crashed process and restart it automatically.

**Readiness probe — "is it ready to correctly handle real work right now?"**

A readiness check is a stricter question: not just "is the process alive,"
but "has it finished all its startup work, and can it correctly handle a
real request *right now*?" If the model hasn't finished loading yet, the
process is technically alive (liveness would say yes), but it genuinely
cannot yet produce a correct prediction — so readiness should say no.

**Why this distinction actually matters in practice**

If you only had a liveness check, an orchestration system might start
sending real user traffic to a server the instant its process starts,
before the model has actually finished loading — resulting in failed
requests during every startup or restart. By checking readiness
*separately*, an orchestration system (or a load balancer) can hold back
real traffic until the readiness check confirms the service is genuinely
ready, routing traffic only once it will actually succeed.

**Where it's used in this project**

- `serving/main.py::health()` implements the liveness probe:
  `GET /health` simply returns a fixed success response, with essentially
  no logic — its only job is to confirm the process itself is responding.
- `serving/main.py::ready()` implements the readiness probe: it checks
  `registry.is_loaded`, returning a normal `200` with model details only if
  the champion model has finished loading, and otherwise explicitly
  returning `503` with `"Model not yet loaded"` — a clear, correct signal
  that the process is alive but genuinely not ready for real prediction
  traffic yet.
- The model is loaded on a background thread at startup
  (`serving/main.py`'s `lifespan()` function), specifically so the server
  can start answering liveness checks immediately, while the readiness
  check correctly reports "not ready" until the background loading actually
  finishes.

---

### 4. Model Hot-Reload — Zero-Downtime Updates

**The problem this solves**

Once a new, better model is trained and promoted to champion, the live
serving API needs to start using it. The naive way to do this is to stop
the server, manually swap in the new model file, and restart it — but this
causes a real gap in service, during which the API can't answer any
requests at all. For a production system, especially one meant to be
retrained automatically every night without a human involved, this kind of
manual downtime isn't acceptable.

**What "hot-reload" means**

Hot-reload means swapping in a new model *while the server keeps running
the whole time*, with no restart and no gap in availability. Requests that
are currently in progress finish using the old model; any new request that
arrives after the swap uses the new model. From the outside, a caller of
the API never notices anything happened at all.

**How this is done safely: a lock**

Because many requests can be arriving and being handled at the same time,
swapping the model needs to be done carefully to avoid a request reading a
"half-swapped," inconsistent state. This project uses a **lock**
(`threading.RLock`) around the model — any code that wants to *read* the
current model, or *replace* it with a new one, must first acquire this
lock. This guarantees the swap happens as one clean, atomic ("all at once")
operation from the perspective of every other piece of code, so no request
can ever see a broken, half-updated model.

**How the new model is actually detected**

Rather than needing something to specifically "tell" the API a new model is
ready, a background thread inside the API periodically checks MLflow, every
5 minutes, asking "what version currently has the champion alias?" If this
version is different from the one currently loaded in memory, the
background thread loads the new one and performs the locked swap described
above — entirely automatically, with no human action required.

**Simple analogy**

Think of a highway sign that shows live traffic conditions, updated by a
control room. The sign doesn't need to be physically taken down and
reinstalled every time the conditions change — the control room just
updates what's shown on the existing sign, and drivers passing by
immediately see the new information, with the sign never actually going
dark in between.

**Where it's used in this project**

- `serving/model_loader.py::ModelRegistry` implements the whole pattern:
  `_load_champion()` loads a model and stores it under `self._lock`;
  `_poll_loop()` runs forever on a background thread, sleeping for
  `POLL_INTERVAL_SECONDS = 300` (5 minutes) between checks, and calls
  `_load_champion()` again whenever it detects the champion alias now
  points to a different version than what's currently loaded.
- The `predict()` method also acquires `self._lock` before reading the
  current model to make a prediction, guaranteeing it never reads a model
  that's in the middle of being swapped out.
- This whole mechanism is what allows the nightly Airflow retraining DAG
  (Tier 4) to fully close the loop automatically: a new champion promoted at
  2 AM will be picked up and hot-loaded into the live API within 5 minutes,
  with zero manual restarts and zero downtime.

---

### 5. Async vs Sync in FastAPI

*(This builds directly on the `async`/`await` concept already introduced at
the end of the Tier 0/1 notes — worth a quick look back if it's not fresh.)*

**Why FastAPI specifically cares about this choice for each endpoint**

FastAPI lets you declare each individual endpoint as either a normal,
regular ("sync") function, or an `async def` function. Picking the right
one for each specific endpoint matters for how well the whole server
performs when many requests arrive around the same time.

**When `async def` is the right choice: I/O-bound work**

If an endpoint mostly spends its time *waiting* — waiting to read a file
from disk, waiting for a network response, waiting on a database — marking
it `async def` lets the server work on *other* incoming requests during that
waiting time, instead of sitting completely idle. This project's `/predict`
endpoint is a good fit for this: it involves loading model/feature data
(disk I/O) and doesn't do anything so CPU-heavy that it would starve other
requests of processing time.

**When `async def` is the wrong choice: CPU-bound work**

If an endpoint instead spends its time doing genuinely heavy computation —
actually training a machine learning model, for example — putting that
directly inside an `async def` function is a mistake. Since `async`/`await`
concurrency all happens on a single thread, a long CPU-heavy task will
completely block that thread, meaning the server can't do anything else at
all — not even answer a simple `/health` check — until that CPU-heavy task
finishes. This defeats the entire purpose of using `async` in the first
place.

**How this project handles the CPU-heavy case correctly: threads, not async**

Instead of running training directly inside an `async def` endpoint, the
retraining endpoints are declared as regular functions and rely on
`threading` explicitly: a `threading.Lock`-protected flag
(`_training_active`) tracks whether a training job is currently running,
and any new retrain request arriving while one is already in progress is
immediately rejected with a `409 Conflict` status code, rather than being
allowed to pile up or accidentally run two heavy training jobs at once.

**Simple analogy**

Think of a single waiter working a restaurant. If a customer just needs
their water refilled (a quick, I/O-like task), the waiter can start that,
walk away to help another table while the water pitcher is being fetched,
and come back to finish — nobody else has to wait idle. But if a customer
asks the waiter to personally go into the kitchen and cook their meal from
scratch (a long, CPU-heavy task), the waiter is now fully occupied and
*cannot* help any other table at all until that's done — no amount of
clever scheduling changes that, because the work itself demands their full,
undivided attention the whole time.

**Where it's used in this project**

- `serving/main.py::predict()` and `::ready()` are both declared `async
  def`, matching their I/O-bound nature (reading feature files, checking a
  loaded-model flag).
- `serving/retrain_router.py::train_agent()` (the `/retrain/train` endpoint)
  is also technically declared `async def` for consistency with the rest of
  the router, but the actual heavy lifting — calling
  `models.rl_agent.train_agent.train(algo)`, which runs real PPO/SAC
  training — is guarded by the `_training_lock` / `_training_active` flag
  described above, specifically to prevent two heavy training jobs from
  ever running at the same time and to give any *concurrent* request a
  clear, immediate `409` response instead of quietly queueing up or
  crashing.

---

## TIER 4 — MLOps (What Separates Juniors From Seniors)

### 1. MLflow — Experiments, Runs, Artifacts, Model Registry

**The problem this solves**

When you're training models, you end up trying many different versions —
different hyperparameters, different tickers, different algorithms. Without
some kind of tracking system, you quickly lose track of which version used
which settings, which one actually performed best, and where the saved
model file for that best version even is. MLflow is a tool that keeps a
record of all of this automatically, so nothing gets lost.

**The four building blocks, explained simply**

- **Experiment** — a named folder for a group of related training attempts.
  For example, `"lstm_forecaster"` is one experiment, `"rl_agent_training"`
  is a different experiment.
- **Run** — one single training attempt inside an experiment. Every time you
  train one model, that's one run. A run records three things:
  - **Params** — the settings you used for that run (learning rate, hidden
    size, number of layers, and so on).
  - **Metrics** — the numbers that came out of that run (validation loss,
    Sharpe ratio, directional accuracy). Metrics can even be tracked *over
    time* during a single run, so you can later plot how a metric changed
    as training progressed.
  - **Artifacts** — actual files produced by the run, most importantly the
    trained model file itself, but also things like plots or logs.
- **Model Registry** — a separate, higher-level list of models that have
  been promoted from "just another run" to "an actual named, versioned
  model that other parts of the system can load and use." Every time a new
  version is registered under the same model name, it gets a new version
  number (version 1, version 2, version 3, and so on), and old versions are
  never deleted — you can always go back and load an older one if needed.

**Alias — "champion"**

Instead of every other part of the system needing to know the exact version
number of the model currently in use (which would need updating every time
a new model is promoted), MLflow lets you attach a text label, called an
**alias**, to a specific version. This project uses the alias `"champion"`.
Any other code that needs the current best model just asks for
`"champion"`, and MLflow automatically points that request at whichever
version currently has that alias — so promoting a new model is just a
matter of moving the alias to a new version number, without changing any
code anywhere else.

**Simple analogy**

Think of MLflow like a lab notebook combined with a filing cabinet. The
notebook records exactly what settings you used and what results you got
for every experiment you ever ran (this is the runs, params, and metrics).
The filing cabinet stores the actual physical results of each experiment
(this is the artifacts — the model files). The "champion" alias is like a
sticky note on the filing cabinet drawer that currently holds your best
result, which you can move to a different drawer any time a better result
comes along.

**Where it's used in this project**

- `models/forecasting/train_forecaster.py` and
  `models/rl_agent/train_agent.py` both call `mlflow.set_experiment(...)`,
  then wrap each training attempt in `with mlflow.start_run(...):`, calling
  `mlflow.log_param(...)`, `mlflow.log_metric(...)`, and
  `mlflow.pytorch.log_model(...)` to record everything about that run.
- The model registry is named `"PortfolioAgent"`
  (`MODEL_NAME = "PortfolioAgent"` in `train_agent.py`), and the
  `"champion"` alias is set using
  `client.set_registered_model_alias(MODEL_NAME, "champion", latest.version)`
  once a new run is judged good enough to be promoted (see the next topic).
- `serving/model_loader.py` loads the model purely by asking for
  `models:/PortfolioAgent@champion` — it never needs to know the actual
  version number, just the alias.

---

### 2. Champion-Challenger Model Promotion

**The problem this solves**

Every time you retrain a model — whether manually, or automatically every
night — you get a brand new candidate model. But a brand new model isn't
automatically better than the one currently being used in production.
Sometimes retraining produces a slightly worse model, just due to random
variation in training (different random starting weights, different random
episodes sampled, and so on). If you blindly swapped in every newly trained
model without checking, your production system's quality could randomly
get worse from one night to the next, for no good reason.

**The champion-challenger pattern, explained simply**

At any given time, there is exactly one model actively serving real
predictions — this is called the **champion**. Any newly trained model is,
at first, just a **challenger** — a candidate that has to prove it's
actually better before it's allowed to take over.

**The rule used in this project: beat the champion by 5%, not just by any
amount**

A challenger isn't promoted just for being marginally better than the
champion (say, by 0.1%) — that tiny an improvement could easily just be
random noise from one training run to another, not a real, meaningful
improvement. Instead, this project requires the challenger's validation
Sharpe ratio to beat the champion's validation Sharpe ratio by at least
**5%** before it's promoted. This creates a buffer against noise, so the
champion only gets replaced when there's a genuinely meaningful improvement,
not just random fluctuation.

**Worked example**

If the current champion has a validation Sharpe of `1.20`, a challenger
needs a validation Sharpe greater than `1.20 × 1.05 = 1.26` to be promoted.
A challenger scoring `1.23` would **not** be promoted — the improvement
isn't large enough to be confident it's real and not just noise. A
challenger scoring `1.35` **would** be promoted.

**Why this prevents "model churn"**

Without this 5% buffer, you could end up in a situation where the champion
changes every single night, back and forth, purely chasing small random
differences between training runs — with no actual improvement happening
over time, just constant, pointless swapping. The 5% threshold keeps the
system stable: the champion only changes when there's a real, meaningful
step forward.

**Where it's used in this project**

- `models/rl_agent/train_agent.py::train()` implements this directly:
  `CHAMPION_IMPROVEMENT_THRESHOLD = 0.05`, and after evaluating a freshly
  trained model, it compares `new_sharpe` against
  `current_champion_sharpe * (1 + CHAMPION_IMPROVEMENT_THRESHOLD)` before
  deciding whether to call `client.set_registered_model_alias(...)` and move
  the `"champion"` alias to the new version.
- The exact same logic is duplicated in
  `serving/retrain_router.py::evaluate_champion()`, which is the endpoint
  Airflow actually calls every night as part of the automated retraining
  process (covered in the next topic).
- If there's no champion yet at all (`current_champion_sharpe == 0.0`), the
  new model is promoted automatically regardless of the threshold — there's
  nothing yet to beat.

---

### 3. Model Drift Detection

**The problem this solves**

A model trained on historical data assumes that the patterns it learned
will keep roughly holding true in the future. But markets change over time
— a strategy that worked well during a calm, trending market might perform
much worse once the market shifts into a choppy, volatile, or fundamentally
different regime. This gradual (or sometimes sudden) decline in a model's
real-world performance, as the world it's operating in changes, is called
**drift**.

**Why you can't just trust a model forever after deploying it**

A model deployed once and left running is essentially frozen in time — it
only knows about the patterns present in the data it was trained on. If the
real world moves on and those patterns stop holding, the model's
predictions gradually become less and less useful, without anyone
necessarily noticing right away unless something is actively watching for
it.

**How this project watches for drift**

The system continuously tracks a **rolling 30-day Sharpe ratio** — the
Sharpe ratio calculated using only the most recent 30 days of live
portfolio returns, updated fresh every time a new day's return comes in.
This gives a moving, up-to-date picture of how well the currently deployed
model is *actually* performing in the real world right now, rather than
relying on the backtest numbers from when it was originally trained (which
only tell you how it performed on *past* data).

**The threshold: 0.5**

If this rolling 30-day Sharpe drops below `0.5`, that's treated as a signal
that something has likely changed in the market and the model may no longer
be performing as well as it was designed to. This threshold is set roughly
one standard deviation below the model's backtested Sharpe (around 1.4–1.47
in this project) — low enough that ordinary day-to-day noise in a 30-day
window won't trigger it constantly (a false alarm), but high enough to catch
a real, meaningful decline in performance before too much damage is done.

**What happens once drift is detected**

Crossing below the threshold doesn't retrain anything by itself — it fires
an **alert**, which then triggers the automated retraining pipeline
(covered in the Airflow topic below) to kick off a fresh round of training
on more recent data, in the hope of producing an updated model that's
better adapted to whatever has changed.

**Where it's used in this project**

- `serving/metrics.py` maintains a rolling window of the last 30 recorded
  daily returns (`deque(maxlen=30)`), and `_update_rolling_sharpe()`
  recalculates the Sharpe ratio from that window every time a new return
  comes in, publishing it as the Prometheus gauge
  `portfolio_rolling_sharpe_30d`.
- Grafana (the dashboard/alerting tool sitting on top of Prometheus) is
  configured to watch this specific gauge and fire a webhook alert when it
  drops below `0.5`, which is picked up by
  `mlops/monitoring/alert_trigger.py` and used to trigger the Airflow
  retraining DAG (see next topic).

---

### 4. Airflow as Orchestrator — Not a Training Environment

**The problem this solves**

Apache Airflow is a tool for scheduling and coordinating multi-step
workflows — running task A, then once it succeeds running task B, and so
on, on a fixed schedule (like "every weekday evening"), with automatic
retries if something fails. A very tempting, but ultimately problematic,
way to use it would be to write the actual model training code directly
inside Airflow's own tasks.

**Why putting heavy ML code directly inside Airflow is a common mistake**

Doing this drags every ML library the training code depends on — PyTorch,
Stable-Baselines3, Gymnasium, MLflow, and so on — directly into Airflow's
own environment. This causes a few concrete problems:

- The Airflow container's image becomes enormous (multiple gigabytes),
  since it now bundles a full deep-learning stack.
- Dependency conflicts become more likely — Airflow itself has its own
  Python package requirements, and cramming a whole separate ML stack into
  the same environment increases the chance that some package version
  clashes with something Airflow needs.
- The scheduler itself, whose main job is just reliably triggering tasks on
  time, becomes less stable, since it's now sharing resources with actual
  heavy model training work.

**The pattern this project uses instead: Airflow as a pure HTTP caller**

Airflow's DAG in this project contains **zero machine learning code**.
Every single task is nothing more than an HTTP request — specifically, a
`requests.post()` call to one of the FastAPI service's `/retrain/*`
endpoints. All the actual heavy lifting (loading data, training PPO/SAC,
evaluating, promoting a champion) happens inside the FastAPI service, which
already has the full ML stack installed, since it also needs those same
libraries to serve real-time predictions.

**Why this separation is a good design choice**

- The Airflow container stays small and lightweight (under 500 MB), needing
  only simple libraries like `requests` to make HTTP calls.
- If the API container happens to restart partway through a long training
  run, the training itself keeps running inside that container
  independently — Airflow just polls a status endpoint later to check how
  it went, rather than the training being tightly coupled to Airflow's own
  process lifecycle.
- Scheduling (Airflow's job) and execution (the API container's job) are
  cleanly separated — you could, in principle, completely swap out the
  scheduler for a different tool without touching any of the actual
  training code at all, since the API's `/retrain/*` endpoints don't know
  or care that Airflow is the thing calling them.

**The nightly workflow, step by step**

```
fetch_new_data  →  engineer_features  →  [train_ppo, train_sac in parallel]  →  evaluate_champion  →  notify_result
```

Every one of these boxes is simply an HTTP call to a different
`/retrain/*` endpoint on the FastAPI service.

**Where it's used in this project**

- `mlops/airflow/dags/retrain_dag.py` defines the DAG above using Airflow's
  `PythonOperator`, where each Python function
  (`task_fetch_data`, `task_engineer_features`, `task_train_ppo`,
  `task_train_sac`, `task_evaluate_champion`, `task_notify_result`) does
  nothing except call `call_training_service()`, a thin wrapper around
  `requests.post(...)`.
- `serving/retrain_router.py` defines the actual `/retrain/fetch-data`,
  `/retrain/engineer-features`, `/retrain/train`, and
  `/retrain/evaluate-champion` endpoints that receive these calls and run
  the real work — this file lives inside the `serving/` folder (the FastAPI
  app), not inside `mlops/airflow/`, confirming where the real ML logic
  actually lives.
- The DAG is scheduled with `schedule = "30 12 * * 1-5"` — 12:30 UTC, which
  is 18:00 IST, Monday through Friday.

---

### 5. Prometheus Pull Model vs. Push Model

**The two different ways a monitoring system can collect metrics**

There are two fundamentally different approaches a monitoring system can
take to gather metrics from your running services:

- **Push model** — each service is responsible for actively *sending* its
  own metrics out to a central monitoring server, usually on some fixed
  schedule (for example, "every 10 seconds, send my current metrics to the
  monitoring server").
- **Pull model** — the monitoring server instead reaches *out* to each
  service on its own schedule and asks, "what are your current metrics
  right now?" The service doesn't initiate anything; it just passively sits
  there, ready to answer whenever asked.

**Prometheus specifically uses the pull model**

Prometheus works by periodically sending an HTTP request to a specific URL
path (by convention, `/metrics`) on each service it's configured to watch.
The service being monitored doesn't send anything out on its own — it just
needs to expose that one endpoint, which returns its current metric values
in a simple, plain-text format whenever it's asked.

**Why this is a meaningfully different approach from ordinary logging**

Regular application logs are typically *pushed* — your application writes
log lines out to a file or log-collection service as things happen. The
pull model flips this: instead of the application deciding when to report
data, the *monitoring system* decides when to check in, at a predictable,
centrally-controlled interval, and the application just needs to always
have its latest numbers ready whenever asked.

**A practical advantage of the pull model**

Because Prometheus is the one deciding when to check each service, it can
easily tell whether a service has gone completely silent or crashed —
simply by noticing that its regular attempts to reach that `/metrics`
endpoint start failing or timing out. With a push model, if a service
crashes, it obviously can't push anything anymore, but it's harder for the
monitoring system to distinguish "this service is fine but just happens to
have nothing new to report right now" from "this service has actually
crashed" without some extra signal.

**Where it's used in this project**

- `serving/metrics.py::metrics_endpoint()` exposes a `GET /metrics` route on
  the FastAPI app that returns all current Prometheus metric values (using
  the `prometheus_client` library's `generate_latest()` function) in the
  plain-text format Prometheus expects.
- `mlops/monitoring/prometheus.yml` is the configuration that tells
  Prometheus *where* to pull from and *how often*:
  ```yaml
  scrape_interval: 15s
  scrape_configs:
    - job_name: "portfolio-api"
      static_configs:
        - targets: ["api:8000"]
      metrics_path: "/metrics"
  ```
  This tells Prometheus to send an HTTP request to `api:8000/metrics` every
  15 seconds, pulling in the four metrics defined in `serving/metrics.py`
  (request count, request latency, daily portfolio return, and the rolling
  30-day Sharpe ratio used for drift detection above).

---

## TIER 5 — Backend and Serving

### 1. REST API Design — Endpoints, HTTP Methods, Status Codes

**What a REST API is, in plain terms**

A REST API is a set of rules for how one piece of software (a client, like
the Streamlit dashboard, or Airflow, or a person testing with curl) can ask
another piece of software (the FastAPI server) to do something, or to give
back some information, using standard web requests over HTTP — the same
underlying technology that powers ordinary web browsing.

**Endpoints**

An endpoint is simply one specific URL path the server understands and
knows how to respond to — for example, `/predict`, `/health`, or `/metrics`.
Each endpoint typically does one clear, specific job.

**HTTP methods — the "verb" of a request**

Every request to an endpoint also specifies a method, which describes the
*kind* of action being requested:

- **GET** — "give me some information, don't change anything." Used for
  reading data. In this project: `/health`, `/ready`, and `/metrics` are all
  `GET` requests, since they only report information back and don't change
  any state on the server.
- **POST** — "here is some data, please do something with it, likely
  causing some action or change." In this project: `/predict` is a `POST`
  request, because the client is sending a list of tickers and asking the
  server to run a computation and produce a result; `/retrain/train` is
  also `POST`, since it kicks off an actual training run.

**Status codes — a short numeric summary of what happened**

Every response from the server comes back with a numeric status code
summarising the outcome:

- **200** — success, everything worked as expected.
- **401** — unauthorised, meaning the request didn't include valid
  credentials (in this project, a missing or wrong API key).
- **422** — the request was understood, but the data sent didn't pass
  validation (for example, asking for a prediction with only 1 ticker when
  at least 2 are required).
- **500** — something went wrong inside the server itself while trying to
  handle an otherwise valid request (an unexpected internal error).
- **503** — the service is temporarily not ready to handle this kind of
  request right now (in this project, this happens if a prediction is
  requested before the model has finished loading at startup).

**Why this standardisation is useful**

Any client talking to this API, and any developer debugging it, can look at
just the method and status code and immediately understand roughly what
happened, without needing to read through custom, one-off documentation for
every single endpoint — GET vs POST tells you whether something was
changed, and the status code tells you, at a glance, whether it succeeded,
and if not, roughly what category of problem occurred.

**Where it's used in this project**

- `serving/main.py` declares each endpoint with its specific method, for
  example `@app.get("/health", ...)` and `@app.post("/predict", ...)`.
- Status codes are raised explicitly wherever something goes wrong:
  `raise HTTPException(status_code=401, ...)` for a bad API key,
  `raise HTTPException(status_code=422, ...)` if `predict_weights()` raises
  a `ValueError` (for example, too few valid tickers), and
  `raise HTTPException(status_code=503, ...)` if the model hasn't finished
  loading yet.

---

### 2. Pydantic — Data Validation at API Boundaries

**The problem this solves**

When a client sends data to your API (for example, a list of stock tickers
to get a prediction for), you can't just trust that the data will always be
exactly the shape and type you expect. A client might send an empty list,
a list with only one ticker when two are required, a ticker with weird
whitespace or lowercase letters, or even completely malformed data. If you
don't check this carefully, badly-shaped data can silently work its way
deep into your system before eventually causing a confusing crash somewhere
far from where the actual problem originated.

**What Pydantic does**

Pydantic lets you define exactly what a valid request (or response) should
look like, using a simple class with named fields and their expected types.
FastAPI automatically checks every incoming request against this definition
*before* any of your actual business logic runs. If the incoming data
doesn't match, FastAPI automatically rejects the request with a `422`
status code and a clear description of exactly what was wrong — your own
code never even has to deal with badly-shaped input, because it's already
been filtered out before it gets there.

**Worked example from this project**

```python
class PredictRequest(BaseModel):
    tickers: list[str] = Field(..., min_length=2, max_length=50)
    date: Optional[str] = Field(default=None)
```

This single class definition automatically enforces several rules at once:
`tickers` must be present (the `...` means required, no default value),
must be a list of text strings, and that list must contain at least 2 and
at most 50 items. `date` is optional — if the client doesn't include it,
it simply defaults to `None`, and the rest of the code knows to treat a
missing date as "use the latest available data."

**Field validators — custom rules beyond just type and length**

Sometimes you need a rule more specific than "must be a list of strings
between 2 and 50 items long." Pydantic lets you attach a custom validator
function that runs extra checks (and can even clean up the data) before it's
accepted.

```python
@field_validator("tickers")
@classmethod
def tickers_non_empty_strings(cls, v: list[str]) -> list[str]:
    for t in v:
        if not t.strip():
            raise ValueError("Ticker symbols cannot be empty strings")
    return [t.strip().upper() for t in v]
```

This checks that no ticker in the list is just empty whitespace, and also
**cleans up** every ticker automatically — removing extra spaces and
converting to uppercase — so that later code never has to worry about
receiving a messy ticker like `"  reliance.ns "` instead of the clean
`"RELIANCE.NS"` it actually wants.

**Response models — validation goes both ways**

Pydantic isn't only used for checking incoming requests — it's also used to
define the *shape of the response* the API sends back
(`PredictResponse`, `HealthResponse`, `ReadyResponse`). This guarantees the
API always returns data in a predictable, consistent structure, and it's
also what powers FastAPI's automatic interactive documentation page
(`/docs`), since FastAPI can read these class definitions to automatically
describe exactly what every endpoint expects and returns.

**Where it's used in this project**

- `serving/schemas.py` defines all four schema classes:
  `PredictRequest`, `PredictResponse`, `HealthResponse`, and
  `ReadyResponse`, each with clear field types, descriptions, and, where
  needed, extra validation rules.
- `serving/main.py`'s `/predict` endpoint takes `body: PredictRequest` as a
  plain function argument — FastAPI handles all the checking and parsing
  automatically before the function body even starts running.

---

### 3. Liveness vs. Readiness Probes

**The problem this solves**

When you're running a service (especially one managed by an orchestration
tool like Docker or Kubernetes), you often need to answer two subtly
different questions: "is this service's process still running at all?" and
"is this service currently able to properly handle real work?" These sound
similar but are genuinely different questions, and conflating them can
cause real problems.

**Liveness — "is the process alive?"**

A liveness check answers a very basic question: has the process crashed, or
is it hung in some completely broken state? If a liveness check fails
repeatedly, the usual response from an orchestration system is to simply
restart the process, assuming something has gone badly wrong internally.

**Readiness — "is the service ready to actually do its job right now?"**

A readiness check answers a more specific question: even though the process
itself is technically running fine, is it currently in a state where it can
properly handle real requests? A service can be perfectly "alive" (the
process hasn't crashed) while still not being "ready" — for example, right
after startup, before it's finished loading a large model into memory.

**Why conflating these two checks causes real problems**

If you only had one combined health check, and it always just answered
"yes, alive" the instant the process starts (even before the model is
loaded), traffic could start being routed to the service before it's
actually capable of handling it properly — leading to a wave of failed
requests right after every restart or deployment, until the model finishes
loading. Separating the two checks lets an orchestration system know: "keep
this process running, don't restart it (it's alive) — but also, don't send
it real traffic just yet (it's not ready)."

**How this project implements the distinction**

- `GET /health` always returns `200 OK` immediately, as long as the FastAPI
  process itself is running at all — it doesn't check anything about the
  model.
- `GET /ready` specifically checks whether the trained model has finished
  loading into memory. If it hasn't finished loading yet, `/ready` returns
  `503` (service unavailable) instead of `200`, even though `/health` would
  still be returning `200` at the very same moment.

**Real-world example of why this matters**

Imagine the API container restarts (say, after a deployment of new code).
The process starts up instantly, so `/health` immediately returns `200`.
But loading the trained RL model from disk, plus loading all the feature
data, takes a few seconds. During those few seconds, if a load balancer
were only checking `/health`, it would start sending real prediction
requests to a container that isn't ready yet, and those requests would fail
with a confusing error. By checking `/ready` instead (or in addition), the
load balancer knows to hold off sending real traffic until the model has
actually finished loading.

**Where it's used in this project**

- `serving/main.py::ready()` checks `registry.is_loaded` (a simple boolean
  flag on the `ModelRegistry` singleton) and returns a `503` with a
  `JSONResponse` if the model hasn't finished loading, or `200` with model
  details (version, algorithm) if it has.
- `serving/main.py::health()` does no such check at all — it simply always
  returns a fixed `HealthResponse()` with `status="ok"`, confirming only
  that the process itself is up and responding.
- The model is deliberately loaded on a **background thread** at startup
  (`threading.Thread(target=load_in_background, daemon=True).start()`
  inside the `lifespan` function), specifically so the server can start
  answering `/health` checks (and generally accepting connections)
  immediately, while `/ready` correctly continues to report `503` in the
  background until loading actually finishes.

---

### 4. Model Hot-Reload — Zero-Downtime Updates

**The problem this solves**

Once a new champion model has been promoted (see the champion-challenger
topic above), the running API server somehow needs to start using that new
model instead of the old one. The most obvious way to do this would be to
simply restart the server — but a full restart means the service is
completely unavailable for some period of time while it shuts down and
starts back up, which is disruptive for anyone trying to use the API during
that window, especially for a service that's meant to be running
continuously.

**What "hot-reload" means**

Hot-reload means swapping in the new model **while the server keeps
running continuously**, without any restart, and without any period where
the service stops responding to requests at all. From the outside, a client
calling the API notices nothing different — one moment their prediction
uses the old model, shortly after, it uses the new one, with zero
interruption in between.

**How this project achieves it: a background polling thread**

A separate background thread, running continuously alongside the main
server, periodically (every 5 minutes, in this project) checks the MLflow
registry to see if the `"champion"` alias now points to a different model
version than the one currently loaded in memory. If it detects a change, it
loads the new model into memory, and only then, swaps it in to replace the
old one.

**Why the swap step needs a lock**

The main server could be actively in the middle of handling a prediction
request, using the current model, at the exact same moment the background
thread finishes loading a new model and wants to swap it in. Without any
protection, this could cause a request to end up reading a half-updated,
inconsistent state — for example, seeing a new model in one line of code
but old, unrelated internal state left over from the previous model in
another line. A **lock** (specifically, an `RLock`, or "reentrant lock")
ensures that the swap happens as a single, uninterrupted, all-or-nothing
step: any in-flight request that started using the old model finishes
cleanly using the old model, and only requests that start *after* the swap
is fully complete will see the new model. No request ever sees a
half-swapped, inconsistent mixture of the two.

**Real-world example**

This is similar to how you'd want to replace a road sign at a busy
intersection — you wouldn't want drivers to see the old sign halfway
covered by the new one while it's being installed, causing confusion. You'd
either swap it instantly in one motion, or briefly cover it entirely during
the swap, so every driver at any given moment sees one clear, consistent
sign — never an ambiguous, half-changed state.

**Where it's used in this project**

- `serving/model_loader.py::ModelRegistry` is the class that implements all
  of this: `_load_champion()` loads a model from MLflow and, under
  `self._lock`, atomically updates `self._policy`, `self._version`, and
  `self._algo` all together. `_poll_loop()` runs forever on a background
  thread (`POLL_INTERVAL_SECONDS = 300`, meaning every 5 minutes), comparing
  the currently loaded version against the registry's current
  `"champion"` alias, and calling `_load_champion()` again only if they
  differ.
- This background thread is started once at server startup via
  `_start_polling()`, called from inside `registry.load()`, which itself is
  kicked off from the `lifespan` function in `serving/main.py`.

---

### 5. Async vs. Sync in FastAPI

**Quick recap of `async`/`await` (covered under Tier 0/1's bonus section)**

An `async def` function can pause itself while waiting on something slow
(like reading a file, or a network call), letting the server go handle a
*different* request in the meantime, and then resume automatically once the
slow thing finishes — all on a single thread. This is great for work that
spends a lot of time simply *waiting*, and not so useful for work that's
genuinely *computing* the whole time.

**Why the `/predict` endpoint in this project is declared `async def`**

Handling a prediction request mostly involves things like reading feature
data and running a fast, already-loaded model to produce an output. While
none of these are extremely slow individually, declaring the endpoint as
`async def` still lets the server remain free to accept and start handling
other incoming requests (like a `/health` check, or another `/predict`
call) rather than being forced to fully finish one request before it can
even begin looking at the next one.

**Why real training work is deliberately kept synchronous instead**

Actually training a PPO or SAC agent is a **genuinely CPU-heavy** task that
can run for a long time (the `/retrain/train` endpoint allows up to 2 hours,
per the Airflow DAG's `TIMEOUT_TRAIN` setting). If this heavy computation
were placed directly inside an `async def` function and run on the
server's single event-loop thread, it would completely block that thread
for the entire duration of training — meaning the server would become
totally unresponsive to *any* other request, including simple health
checks, for potentially hours at a time. This defeats the entire purpose of
using `async` in the first place.

**How this project actually protects against that**

Rather than relying on `async`/`await` for the training endpoint, the
project instead uses a simple **lock-based flag** to track whether training
is currently in progress at all, and rejects new training requests outright
while one is already running, rather than trying to run multiple trainings
concurrently or interleave training with other request handling on the same
thread.

```python
_training_lock   = threading.Lock()
_training_active = False

@retrain_router.post("/train")
async def train_agent(algo: str = "ppo", ...):
    if _is_training():
        raise HTTPException(status_code=409, detail="Training already in progress")
    _set_training(True)
    try:
        train(algo)          # the actual, long, CPU-heavy training call
    finally:
        _set_training(False)
```

The endpoint function itself is still written as `async def` (a FastAPI
convention), but the actual heavy `train(algo)` call inside it is an
ordinary, blocking, synchronous function call — the project accepts that a
training request will occupy the server for its full duration, and instead
guards against the *specific* danger of two trainings overlapping and
fighting over the same resources at once, using the lock and the `409
Conflict` status code ("something about the current state of the server
conflicts with this request") to cleanly reject a second training request
if one is already running.

**The general rule worth remembering**

Use `async def` freely for endpoints whose work is dominated by *waiting*
(reading files, calling other services, waiting on a database). Be careful
with, and generally avoid putting, genuinely heavy *computation* directly
inside an `async def` function on the main server thread — that kind of
work should either be handled with an explicit understanding that it will
occupy the server for its duration (as done here, with a lock preventing
overlap), or be handed off to a separate worker process entirely so the
main server stays responsive no matter how long the heavy work takes.

**Where it's used in this project**

- `serving/main.py::predict()` and `serving/main.py::health()` /
  `serving/main.py::ready()` are all declared `async def`, appropriate for
  their relatively light, I/O-leaning work.
- `serving/retrain_router.py::train_agent()` is also technically
  `async def` (a FastAPI requirement for route handlers), but its actual
  work — calling `train(algo)` — is a long-running, synchronous, CPU-bound
  call, deliberately protected from overlapping with itself using the
  `_training_lock` / `_training_active` flag described above, rather than
  being made genuinely non-blocking.

---

## TIER 6 — Infrastructure

### 1. Docker Images, Containers, and Layers

**The problem this solves**

"It works on my machine" is one of the oldest problems in software. A
project might depend on a specific version of Python, a specific set of
installed libraries, and specific system-level tools. If a teammate, or a
production server, has even slightly different versions of any of these,
the project can behave differently or simply fail to run at all. Docker
solves this by packaging the application together with everything it needs
to run — the exact Python version, all libraries, all system tools — into
one single, self-contained unit that behaves the same way no matter where
it's run.

**Image vs. container — the key difference**

- An **image** is a saved, read-only blueprint — a snapshot of a filesystem
  containing the application code, installed libraries, and system tools,
  along with instructions on how to start the application. An image by
  itself isn't running anything; it's just a file sitting on disk (or in a
  registry, like Docker Hub), ready to be used.
- A **container** is a running instance created *from* an image. You can
  start multiple separate containers from the very same image, and each one
  runs independently, with its own memory and process space, but all
  starting from that exact same blueprint.

**Simple analogy**

An image is like a recipe (or a cake mould), and a container is like an
actual cake baked using that recipe. You can bake many cakes (containers)
from the same recipe (image), and each one comes out identical to start
with, even if they then go on to have separate lives afterward (one cake
gets eaten, another gets frozen).

**Layers — why images are built the way they are**

A Docker image isn't one giant blob — it's built up from a stack of
**layers**, where each layer represents one instruction in the file used to
build the image (called a `Dockerfile`). For example, one layer might be
"install Python," the next layer "install these specific libraries," the
next layer "copy in the application's source code." Docker is smart about
reusing layers: if you rebuild an image and only the very last layer
actually changed (say, you only edited a source code file, without changing
any library requirements), Docker can reuse all the earlier, unchanged
layers instead of redoing all that work again, making rebuilds much faster.

**Why the order of instructions in a Dockerfile matters**

Because of this layer-reuse behaviour, it's a common best practice to put
instructions that change *rarely* (like installing libraries, which usually
only changes when you add a new dependency) *before* instructions that
change *often* (like copying in your actual application source code, which
changes every time you edit a file). This way, editing your source code
doesn't force Docker to redo the much slower "install libraries" step every
single time, since that layer stays unchanged and gets reused from a
previous build.

**Where it's used in this project**

- `serving/Dockerfile` copies in `requirements/api.txt` and runs
  `pip install` *before* copying in the actual application code
  (`data/`, `models/`, `serving/`, `backtesting/`) — exactly the ordering
  described above, so that editing application code doesn't force a full
  reinstall of every Python library on the next build.
- The base image used is `python:3.11-slim-bullseye` — a pre-built image
  that already has Python 3.11 installed on a small ("slim") version of the
  Debian Linux distribution, so the project doesn't need to install Python
  itself from scratch.

---

### 2. Multi-Stage Builds

**The problem this solves**

Building certain Python packages from source sometimes needs extra tools —
compilers like `gcc` and `g++`, for example — which are only needed
*during* the installation step itself, not while the application is
actually running afterward. If you install these compiler tools into your
final image and never remove them, your image ends up permanently larger
than it needs to be, uses more disk space, takes longer to download and
deploy, and unnecessarily includes extra tools that could, in principle,
also increase what an attacker could potentially do if they ever got access
to the running container.

**What a multi-stage build is**

A multi-stage build lets a single `Dockerfile` define more than one
temporary "stage," where an early stage can do all the heavy, messy work
(like compiling packages using `gcc`), and a later, final stage only copies
across the specific *finished results* it actually needs from the earlier
stage — leaving behind everything else, including all the temporary
compiler tools, which simply never make it into the final image at all.

**Worked example, following this project's own Dockerfile**

```dockerfile
# Stage 1: builder — has compilers, used only to install packages
FROM python:3.11-slim-bullseye AS builder
RUN apt-get install -y gcc g++ curl git
COPY requirements/api.txt ./
RUN pip install --prefix=/install -r api.txt

# Stage 2: runtime — the actual final image, no compilers included
FROM python:3.11-slim-bullseye AS runtime
COPY --from=builder /install /usr/local
```

The first stage (`builder`) installs `gcc`, `g++`, and other build tools,
and uses them to install the Python packages into a specific folder
(`/install`). The second stage (`runtime`) starts completely fresh from a
clean base image, and the only thing it takes from the first stage is that
`/install` folder — the actual finished, compiled packages. The `gcc`,
`g++`, `curl`, and `git` tools used during the build are simply left behind
in the discarded first stage and never appear in the final image at all.

**Simple analogy**

Think of it like building furniture: the workshop is full of saws, drills,
sanders, and sawdust everywhere (the builder stage). Once the piece of
furniture is actually finished, you don't ship the entire workshop to the
customer — you just ship the finished chair (the runtime stage), leaving
all the tools and mess behind.

**Where it's used in this project**

- `serving/Dockerfile` is exactly the two-stage pattern shown above — the
  final `runtime` stage only installs `curl` (needed for the `HEALTHCHECK`
  instruction, described below), while `gcc`, `g++`, and `git` from the
  `builder` stage never appear in the final image, keeping it smaller.

---

### 3. `docker-compose` — Multi-Container Orchestration

**The problem this solves**

This project isn't just one single container — it's made up of several
separate, cooperating services: the FastAPI backend, the Streamlit
dashboard, MLflow, Prometheus, Grafana, and three separate Airflow
components (init, webserver, scheduler). Manually starting each of these
one at a time, with the right settings, in the right order, and making sure
they can all talk to each other, would be tedious and very easy to get
wrong. `docker-compose` solves this by letting you describe *all* of these
services, and how they relate to each other, in a single configuration
file, and then start (or stop) the entire group with one single command.

**Services**

Each entry under `services:` in the `docker-compose.yml` file is one
container definition — what image or Dockerfile to use to build it, what
ports it needs, what environment variables it needs, and so on. This
project defines 8 services in total.

**`depends_on` — controlling startup order**

The `depends_on` setting tells Docker Compose that one service should only
be started *after* another service has already started. For example, the
`api` service lists `depends_on: [mlflow]`, meaning Docker Compose will
make sure the `mlflow` container has already been started before it starts
the `api` container — this makes sense, since the API needs to be able to
reach MLflow to load the trained model.

**Networking between containers — using service names, not `localhost`**

When multiple containers are started together via `docker-compose`, they
can all reach each other over the network using their *service name* as if
it were a hostname — for example, the `api` service connects to MLflow
using the address `http://mlflow:5000`, not `http://localhost:5000`. Docker
Compose automatically sets up an internal network where the name `mlflow`
resolves correctly to the actual `mlflow` container, purely because that's
the name given to it in the `services:` section.

**Volumes — keeping data even when a container is removed**

A container's own internal filesystem is, by default, temporary — if you
delete and recreate a container, anything it wrote to its own internal disk
is lost. A **volume** is a way of connecting a folder on the actual host
machine (or a Docker-managed storage area) to a folder inside the
container, so that data written by the container is actually saved
*outside* the container itself, and survives even if the container is
deleted and a brand new one is started in its place.

**Where it's used in this project**

- `docker-compose.yml` defines these services: `mlflow`, `api`,
  `dashboard`, `prometheus`, `grafana`, `airflow-init`, `airflow-webserver`,
  and `airflow-scheduler`.
- Volumes like `./mlruns:/app/mlruns` and
  `./models/rl_agent/saved_models:/app/models/rl_agent/saved_models` mean
  the API container's saved models and MLflow's run history live on the
  actual host machine's disk, so a trained model isn't lost if the `api`
  container is ever removed and recreated.
- `prometheus_data`, `grafana_data`, and `airflow_db` are declared under a
  top-level `volumes:` section without a specific host folder path
  (`prometheus_data:` with nothing after the colon) — this is a
  Docker-*managed* volume rather than one tied to a specific folder you
  chose yourself, letting Docker decide exactly where on the host machine
  to actually store this data.
- `depends_on` chains reflect the real startup order needed:
  `dashboard` depends on `api`; `prometheus` and `airflow-init` depend on
  services being reachable before they start; `airflow-webserver` and
  `airflow-scheduler` both depend on `airflow-init` (which sets up
  Airflow's own internal database first) and on `api` (since Airflow's
  whole job is calling the API's `/retrain/*` endpoints).

---

### 4. CI vs. CD — What Each Actually Does

**The general idea**

CI/CD is shorthand for two related but distinct automated practices that
happen every time code changes are pushed to a shared repository.

**CI — Continuous Integration**

CI means automatically checking that new code changes are actually good —
running things like linters (tools that check code style and catch obvious
mistakes), automated tests, and build checks — *every single time* someone
pushes a change, rather than only occasionally or manually. The goal is to
catch problems as early as possible, right when a change is made, instead
of discovering them much later after the broken code has already been
merged and possibly deployed.

**CD — Continuous Deployment (or Delivery)**

CD means automatically taking code that has already passed all its CI
checks and actually shipping it somewhere real — building a deployable
package (in this project, a Docker image) and pushing it to a place where
it can actually be run in production, all without a human needing to
manually perform those steps by hand each time.

**Why they're kept as two separate workflows in this project**

Separating CI and CD lets each one run under different conditions, for good
reasons. CI should run on *every* push to *any* branch (`branches: ["**"]`
in this project) — you want to know immediately if a change breaks
anything, no matter which branch someone is working on. CD, by contrast,
should typically only run once code has been merged into the main,
trusted branch (`branches: [main]` in this project) — you don't want every
experimental change on every side branch to automatically get built and
pushed out as a deployable image.

**This project's CI workflow, step by step**

1. **Lint** — `ruff check .` scans the codebase for style problems and
   obvious mistakes, without actually running any of the code.
2. **Unit tests** — runs test files that don't need real market data
   (`test_data_pipeline.py`, `test_forecasting.py`, `test_env.py`,
   `test_agent.py`, `test_backtesting.py`, `test_monitoring.py`).
3. **API integration tests** — a separate step running `test_api.py`,
   which tests the FastAPI endpoints using mocked (fake, stand-in) data
   rather than a fully real, running system.
4. **Docker build check** — actually builds both the API and dashboard
   Docker images from the project's Dockerfiles, to catch any problem where
   the code passes all its tests but the Docker image itself somehow fails
   to build.
5. **Smoke tests** — starts an actual container from the freshly built
   image and repeatedly checks its `/health` endpoint (for the API) or
   `/_stcore/health` endpoint (for the dashboard) for up to 60 seconds,
   confirming the container doesn't just build successfully, but also
   actually starts up and responds correctly once running.

**This project's CD workflow, step by step**

1. Logs in to Docker Hub using credentials stored securely as GitHub
   Secrets (covered in the next topic).
2. Builds the API Docker image and the dashboard Docker image.
3. Tags each image with several different tags at once — a tag based on the
   specific git commit hash, a tag based on the branch name, and a generic
   `latest` tag — so the exact same built image can be referred to in
   multiple useful ways later.
4. Pushes both finished images up to Docker Hub, making them available to
   be pulled down and run on an actual server.

**Where it's used in this project**

- `.github/workflows/ci.yml` runs on every push to any branch and on every
  pull request targeting `main`, containing the `lint-and-test` and
  `docker-build` jobs described above.
- `.github/workflows/cd.yml` runs only on pushes to `main` (or when
  manually triggered via `workflow_dispatch`), containing the
  `build-and-push-api` and `build-and-push-dashboard` jobs.
- The CD workflow deliberately depends on the CI checks having already been
  configured as required checks on the `main` branch (a GitHub repository
  setting, not something visible directly inside the workflow file itself)
  — meaning, in a properly configured repository, code isn't normally
  allowed to even reach `main` unless it already passed CI first, so by the
  time CD runs, the code has already been validated.

---

### 5. GitHub Secrets — Managing Credentials in CI

**The problem this solves**

The CD workflow needs to log in to Docker Hub, which requires a username
and a password (or an access token). But the workflow file itself
(`cd.yml`) is just a plain text file, checked into the Git repository,
which means it's visible to anyone who can view the repository. If real
credentials were typed directly into this file, anyone browsing the
repository's code — including, for a public repository, literally anyone on
the internet — would immediately have full access to those credentials.

**What GitHub Secrets are**

GitHub Secrets are a separate, protected storage area, configured
separately in the repository's own settings (not inside any file that gets
committed to the code itself). A workflow file can *reference* a secret by
name, using special syntax like `${{ secrets.DOCKER_HUB_TOKEN }}`, and
GitHub will substitute in the real, actual secret value only at the moment
the workflow actually runs — the real value is never written into the
workflow file itself, and it's never shown in any logs, build output, or
anywhere else visible to someone just reading the repository's code.

**How this project uses secrets**

```yaml
- name: Log in to Docker Hub
  uses: docker/login-action@v3
  with:
    username: ${{ secrets.DOCKER_HUB_USERNAME }}
    password: ${{ secrets.DOCKER_HUB_TOKEN }}
```

Anyone reading this file can see *that* a username and password are being
used, and can see the *names* given to these secrets
(`DOCKER_HUB_USERNAME`, `DOCKER_HUB_TOKEN`), but cannot see their actual
values anywhere in the file. Those real values were separately typed into
GitHub's own secret-management settings page by whoever administers the
repository, one time, and are never exposed in the code itself.

**A gap worth being aware of in this project**

While Docker Hub credentials are correctly handled as GitHub Secrets, the
`docker-compose.yml` file itself contains a couple of hardcoded, real-
looking secret values directly in plain text for the Airflow webserver and
scheduler services — for example,
`AIRFLOW__CORE__FERNET_KEY=46BKJoQYlPPOexq0OhDZnIlNepKFf87WFwLt0nfdstM=`
and `AIRFLOW__WEBSERVER__SECRET_KEY=portfolio_secret_key_2024`, typed
directly into the file rather than referenced via an environment variable
(`${AIRFLOW_FERNET_KEY}`, which the separate `airflow-init` service in the
very same file correctly uses instead). This inconsistency is worth
noticing and understanding, since it's a genuine, realistic example of the
exact secret-leakage risk that GitHub Secrets and environment-variable
substitution are meant to prevent — good to be able to point out
if reviewing or improving the project further.

**Also worth noting: `serving/Dockerfile` copies `.env` directly into the
image**

The line `COPY .env ./.env` inside `serving/Dockerfile` bakes whatever is
inside the local `.env` file (which typically holds real secrets like API
keys) directly into the built Docker image itself. Anyone who later gets
access to that built image — for example, by pulling it down from Docker
Hub — could potentially extract that `.env` file back out of the image and
read its contents. The safer, more standard approach (which this project
does also use, separately, in `docker-compose.yml` via `env_file: [.env]`)
is to supply environment variables to a *running* container from the
outside, rather than permanently baking a secrets file into the image
itself.

**Where it's used in this project**

- `.github/workflows/cd.yml` references `${{ secrets.DOCKER_HUB_USERNAME }}`
  and `${{ secrets.DOCKER_HUB_TOKEN }}`, which must be configured ahead of
  time in the GitHub repository's Settings → Secrets and variables → Actions
  page.
- `.github/workflows/ci.yml`, by contrast, needs no real secrets at all —
  it deliberately creates a throwaway, fake `.env` file
  (`echo "API_KEY=ci-test-key" > .env`) purely so the Docker image can be
  built and smoke-tested successfully, without ever touching any real
  credentials.

---

## TIER 7 — Interview Questions (Model Answers)

These are the exact questions listed in the roadmap. Each answer below is
written the way you'd actually say it out loud in an interview — short,
direct, and backed by specific numbers and specific files from your own
project, not vague generalities.

### 1. "Why RL for portfolio allocation instead of Markowitz?"

**Short answer**

Markowitz is a good, simple starting point, but it has real weaknesses this
project is specifically designed to address, and the RL agent beat it on
every major metric in backtesting.

**The fuller answer**

Markowitz makes a strong simplifying assumption: it treats expected return
and covariance as fixed numbers for a given period, and solves for the
mathematically "optimal" weights *for that one period*, rebalancing
periodically (monthly, in this project's baseline) using a fresh estimate
each time. It has no real concept of *sequential decision-making* — it
doesn't naturally reason about how today's trade affects tomorrow's
opportunities, and it doesn't have any built-in awareness of transaction
costs unless you bolt that on separately.

RL is a genuinely different kind of tool: it's built from the ground up for
sequential decision-making under uncertainty, where every action affects
the state you'll be in next, and where you can bake real-world frictions —
like transaction costs — directly into the thing being optimised (the
reward function), rather than as an afterthought.

Concretely, in this project's walk-forward backtests, the RL agent achieved
a Sharpe ratio around 1.4–1.47, compared to Markowitz's roughly 0.74–0.77 —
almost double. It also had a smaller maximum drawdown in the reported runs.

**A fair, balanced point to add**

It's also honest to say Markowitz has real advantages RL doesn't: it's
much more interpretable (you can directly see *why* it picked a given
weight, from the expected returns and covariance matrix it used), it
doesn't need any training time at all, and it's far more predictable and
easier to reason about mathematically. That's exactly why this project
keeps Markowitz around as a baseline, rather than treating it as
obsolete — it's a legitimate, standard comparison point that shows whether
the far more complex RL approach is actually earning its added complexity.

---

### 2. "How did you prevent look-ahead bias in backtesting?"

**Short answer**

Several layers, all aimed at the same rule: nothing used to make a decision
at time T is allowed to be calculated using any information from after
time T.

**The fuller answer, as a checklist you can walk through**

- **Feature scaling** — the `RobustScaler` used before feeding data into the
  LSTM is fit only on each window's training slice, never on validation or
  test data, and never on the full dataset combined (`dataset.py`).
- **Covariance estimation** — the Ledoit-Wolf covariance matrix used inside
  Markowitz is recalculated fresh at each rebalance date, using only a
  backward-looking lookback window of past returns, never anything from
  after that date (`markowitz.py::simulate()`).
- **RL training episodes** — the environment's random start dates are
  restricted to only fall within the training period
  (`train_end_idx`), so the agent never trains on data from the
  validation or test period, even by accident (`portfolio_env.py::reset()`).
- **Walk-forward structure itself** — rather than one single train/test
  split, the whole backtest rolls forward through many independent windows
  (504 training days, 126 test days, stepping forward 21 days at a time),
  and within each window, the test period is always strictly *after* the
  training period, never overlapping (`walk_forward.py`).
- **Rolling technical indicators** — every feature (momentum, volatility,
  RSI, moving averages) uses standard, trailing (backward-looking) rolling
  windows, never a centered window that would peek into future rows
  (`feature_engineering.py`).

**A good way to close this answer**

The single sentence that ties it together: "every one of these is really
the same underlying rule, applied at a different layer of the system — a
model, a scaler, or an optimiser is never allowed to see, directly or
indirectly, anything that happened after the point in time it's making a
decision for."

---

### 3. "How does the system detect and respond to model drift?"

**Short answer**

A rolling 30-day Sharpe ratio is tracked continuously in production; if it
drops below a threshold of 0.5, that triggers an alert, which kicks off
automatic retraining.

**The fuller answer, as a pipeline you can describe end-to-end**

1. Every day, the live portfolio's actual return gets recorded, and a
   rolling 30-day Sharpe ratio is recalculated from just the most recent 30
   days of real returns (`serving/metrics.py`, published as the Prometheus
   gauge `portfolio_rolling_sharpe_30d`).
2. Prometheus pulls this number from the API's `/metrics` endpoint every 15
   seconds (`mlops/monitoring/prometheus.yml`).
3. Grafana, watching that same number, is configured to fire a webhook
   alert if it drops below 0.5 — a threshold set below the model's original
   backtested Sharpe (around 1.4) to allow for normal day-to-day noise
   without triggering false alarms constantly.
4. That webhook hits `mlops/monitoring/alert_trigger.py`, a small Flask
   service, which responds by running a subprocess command to manually
   trigger the Airflow retraining DAG immediately, rather than waiting for
   its next scheduled nightly run.
5. Airflow's DAG then calls the FastAPI service's `/retrain/*` endpoints in
   sequence — fetch fresh data, engineer features, train new PPO/SAC
   candidates, and evaluate whether either one beats the current champion
   by at least 5% before promoting it.

**Good honest caveat to mention if asked**

It's worth being upfront that this drift-detection loop is fully wired
together in the codebase (metrics tracking, alerting config, and the
retraining DAG all exist and connect correctly), but a live 30-day rolling
Sharpe genuinely reacting to real production traffic hasn't been observed
running for a full 30+ day stretch yet, since this is a portfolio/resume
project rather than a live production deployment — the mechanism is real
and correctly built, but that specific claim is about design, not about
having watched it trigger on live data over time.

---

### 4. "Why Sortino instead of Sharpe as the reward?"

**Short answer**

Because Sharpe penalises *all* volatility equally, including large positive
days, which actively discourages an RL agent from letting winning trades
run — Sortino only penalises downside volatility, removing that perverse
incentive.

**The fuller answer**

If you train an RL agent by directly rewarding it with a Sharpe-based
signal, you're mathematically telling it that a huge, lucky up day and a
huge, painful down day are both equally "bad," since standard deviation
treats both directions the same way. An agent optimising that reward has an
actual incentive to *avoid* outsized positive returns, since they increase
volatility and shrink the Sharpe ratio — which is obviously the opposite of
what you want a trading strategy to do.

Sortino fixes this specifically: its denominator only measures the spread
of *negative* returns, so a strategy that has occasional large positive
days isn't penalised for those days at all — it's only penalised for being
volatile on the *downside*.

The documented result of making this switch in this project was about 40%
more stable training and roughly a +0.3 improvement in validation Sharpe.

**A precise, honest detail worth adding if the interviewer probes further**

The environment's actual moment-to-moment training signal (`step_reward()`)
is simpler than full Sortino — it's the day's log return minus a
transaction-cost penalty, calculated fresh every single step. Full episode-
level Sortino (`sortino_reward()`) is used to *evaluate and score* finished
episodes and to decide champion promotion, not as the dense, per-step
reward the agent directly optimises during training. The reasoning: PPO's
training works best with an immediate, low-noise signal at every step,
while Sortino needs a reasonably large sample of returns to be a
statistically meaningful number, and that's a better fit for judging a
finished episode than for guiding a single time-step decision.

---

### 5. "Why is Airflow lightweight in your architecture?"

**Short answer**

Because Airflow contains zero machine learning code — every task in the DAG
is just an HTTP call to the FastAPI service, which is where all the actual
PyTorch, Stable-Baselines3, and MLflow logic lives.

**The fuller answer**

The tempting, but flawed, approach would be to write model-training code
directly inside Airflow's own tasks. That drags a full deep-learning stack
(PyTorch, Stable-Baselines3, Gymnasium) into Airflow's own container,
making its image huge, increasing the chance of dependency conflicts with
Airflow's own requirements, and making the scheduler itself less reliable,
since it would now be sharing resources with genuinely heavy training work.

Instead, this project's `retrain_dag.py` file contains nothing but
`requests.post()` calls to endpoints like `/retrain/fetch-data`,
`/retrain/train`, and `/retrain/evaluate-champion` on the FastAPI service.
Airflow's job is purely scheduling and sequencing ("run this task, then
that one, retry on failure, notify on completion") — it doesn't know or
care *how* the training actually happens internally, only that it can POST
a request and later check a status endpoint.

The concrete payoff: the Airflow container's image only needs simple
libraries like `requests`, staying well under 500 MB, versus what it would
need if the full ML stack were bundled in directly.

---

### 6. "What is your model serving strategy?"

**Short answer**

A FastAPI service loads the current MLflow "champion" model at startup on a
background thread, serves predictions from an in-memory copy of that
model's `.zip` file for speed, and polls MLflow every 5 minutes to hot-swap
in a newly promoted champion without any downtime or restart.

**The fuller answer, walking through the actual flow**

1. At startup, the server immediately starts responding to `/health`
   requests, but loads the actual trained model on a separate background
   thread, so the process doesn't block while the model is loading
   (`serving/main.py`'s `lifespan` function).
2. A separate `/ready` endpoint specifically reports whether the model has
   finished loading yet, returning `503` until it has — so anything routing
   real traffic to the service knows not to send prediction requests too
   early.
3. For actual predictions, rather than repeatedly downloading the model
   from MLflow on every single request (which would be slow), the service
   loads the champion's Stable-Baselines3 `.zip` file directly from local
   disk (`serving/predictor.py`), using MLflow's registry only to find out
   *which* algorithm (PPO or SAC) is currently the champion.
4. A background thread polls MLflow every 5 minutes
   (`POLL_INTERVAL_SECONDS = 300` in `model_loader.py`) to check whether the
   `"champion"` alias now points at a different model version than the one
   currently loaded in memory. If so, it loads the new model and, under a
   lock, atomically swaps it in, so any in-flight request completes cleanly
   against whichever model it started with, and no request ever sees a
   half-swapped, inconsistent state.
5. All of this is fronted by standard REST conventions — `POST /predict`
   for getting a prediction, API-key authentication, Pydantic request/
   response validation, and Prometheus metrics exposed at `/metrics`.

**Good line to close with**

"The core design goal was zero-downtime updates — a newly promoted champion
model should start serving real predictions automatically, without anyone
needing to manually restart the API or cause any interruption in service."

---

### 7. "What were your final backtest results?"

**Short answer**

Across walk-forward backtesting, the RL agent achieved a Sharpe ratio of
roughly 1.4–1.47, a CAGR of roughly 14–20%, and a maximum drawdown of
roughly -16% to -33%, outperforming both a Markowitz baseline (Sharpe
≈ 0.74–0.77) and a simple Nifty50 buy-and-hold benchmark (Sharpe ≈ 0.61) on
essentially every major metric, with the outperformance versus equal-weight
found to be statistically significant (p < 0.05) using a two-sided t-test
on monthly excess returns.

**A note on how to actually deliver this answer well**

The README and SYSTEM_DESIGN documents in this project report slightly
different exact numbers from different backtest runs (for example, the
significance test's exact p-value differs quite a bit between the two —
0.024 in one place, a much smaller ~4.3×10⁻²² in another) — a completely
normal thing to happen between separate runs with somewhat different
settings or time windows, but worth being upfront and precise about rather
than quoting a single memorised number as if it were exact and fixed. A
strong way to phrase this in an interview: "depending on the specific
backtest run, Sharpe came in around 1.4 to 1.47 for the RL agent, roughly
double the Markowitz baseline's 0.74 to 0.77 — I'd want to pull the exact
run's numbers before quoting a single precise figure, since I ran this
across a few different configurations."

**Why this kind of honesty is actually a strength in an interview**

Being able to say "I have a range, here's roughly why the numbers vary a
bit between runs, and here's specifically where in the code that number
comes from" demonstrates a real, working understanding of the project,
rather than having simply memorised one number off a README without
understanding where it came from or why it might shift slightly between
runs.

---

## Summary — What You've Now Covered

| Tier | Topic Area | Core Files |
|---|---|---|
| 0 | Python & ML Basics | `feature_engineering.py`, `dataset.py`, `data/config.py` |
| 1 | Finance Fundamentals | `metrics.py`, `markowitz.py`, `reward.py` |
| 2 | Deep Learning (LSTM) | `lstm_model.py`, `train_forecaster.py`, `dataset.py` |
| 3 | Reinforcement Learning | `portfolio_env.py`, `reward.py`, `train_agent.py` |
| 4 | MLOps | `model_loader.py`, `train_agent.py`, `retrain_dag.py`, `alert_trigger.py` |
| 5 | Backend & Serving | `main.py`, `schemas.py`, `model_loader.py`, `retrain_router.py` |
| 6 | Infrastructure | `Dockerfile`, `docker-compose.yml`, `ci.yml`, `cd.yml` |
| 7 | Interview Questions | Model answers tying every tier together |

This completes every topic listed in the original roadmap. Good luck with
the interviews — you now have a working, code-grounded answer for every
piece of the stack, not just a surface-level description of what each tool
does.
