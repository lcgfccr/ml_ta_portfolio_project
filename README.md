# Machine-learning ensemble and hyper-parameter optimiser for a technical-analysis trading strategy on the SSE Composite Index

## NON-TECHNICAL EXPLANATION OF YOUR PROJECT

This project builds and evaluates an automated trading strategy for the Shanghai Stock Exchange Composite Index (SSE). It combines several classic technical-analysis rules into a single system and lets a machine-learning model decide how to weight them over time. The code downloads daily price data, computes indicators such as moving averages and volatility bands, and then tests many different parameter combinations. Performance is judged mainly by the Sortino ratio, which rewards upside and penalises downside risk. The goal is not to predict markets perfectly, but to explore whether structured rules plus learning can shape better risk-return profiles.

## DATA

The strategy uses daily OHLCV (Open, High, Low, Close, Volume) data for the SSE Composite Index, downloaded via the `yfinance` Python library using the Yahoo Finance ticker `000001.SS`. Data is pulled from 2010-01-01 to the latest available date at run time. Adjusted close prices are used as the main input for return calculations, while high, low, close, and volume are used to compute technical indicators via TA-Lib.  
- Source: Yahoo Finance, accessed programmatically via `yfinance`.  
- Index: SSE Composite Index (Shanghai Stock Exchange).

## MODEL

The core of the system is a **meta-model ensemble**:

1. Three rule-based technical “sub-strategies” (Trend Strength, Volatility Reversion, Volume Breakout) each output a discrete trading signal (+1, 0, −1).
2. A **logistic regression** model takes these three sub-strategy signals as features and learns to predict the sign of the next day’s return on the index.
3. The predicted probability of a positive return is converted into a long/short position.

Logistic regression is chosen because it is simple, interpretable, and stable under limited feature sets. It effectively learns a dynamic, state-dependent weighting of the three sub-strategies instead of relying on fixed linear weights.

## HYPERPARAMETER OPTIMISATION

Hyperparameters are tuned to maximise the **cross-validated Sortino ratio** of strategy returns. The search space currently includes:

- `adx_threshold` – minimum trend strength for the Trend strategy.  
- `bb_reversion_period` – lookback for Bollinger Bands in the Reversion strategy.  
- `rsi_oversold` – RSI oversold threshold (overbought is set symmetrically).  
- `atr_period` – ATR lookback for volatility estimation.  
- `bb_breakout_period` – Bollinger Band period for breakout compression.  
- `bb_width_thresh` – squeeze threshold for band width.  
- `logC` – `log10(C)` regularisation strength for logistic regression.

A Sobol quasi-random sequence is used to explore this space more uniformly than a naive grid. For each sampled hyperparameter vector, the model is evaluated with **time-series cross-validation**: in each fold, only past data is used to train the ML ensemble, and the Sortino ratio is computed on the held-out future segment. The final score is the mean Sortino across folds, which the optimiser seeks to maximise.

## RESULTS

<img width="835" height="441" alt="output" src="https://github.com/user-attachments/assets/f38c9437-ce4f-4aa6-9557-4ef7be951c5b" />


Empirically, representing the input to the ML model as **three high-level sub-strategy signals** performs better than feeding raw indicators directly as features (e.g., EMA values, RSI, Bollinger Band levels):

- The sub-strategies encode **domain knowledge**: each signal is already a decision about “trend”, “reversion”, or “breakout” rather than a raw numeric indicator. This acts as strong feature engineering and reduces noise.
- Dimensionality is kept very low (three discrete features), which reduces overfitting risk and makes logistic regression’s coefficients more stable across CV folds.
- Each sub-strategy captures a specific **market regime**; the meta-model’s job is only to learn when each regime is useful instead of discovering regimes from scratch.
- The resulting ensemble benefits from the **bias of well-designed trading rules** and the **adaptivity of ML**, leading (in tests) to higher and more robust Sortino ratios than indicator-only feature sets, where the model tends to chase spurious correlations and regime shifts more aggressively.

