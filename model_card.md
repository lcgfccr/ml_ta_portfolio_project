# Model Card

See the [example Google model cards](https://modelcards.withgoogle.com/model-reports) for inspiration.

---

## Model Description

**Input**

The model consumes:

- Daily OHLCV time series for the SSE Composite Index (`000001.SS`), downloaded via `yfinance`.
- Three discrete sub-strategy signals computed from technical indicators (via TA-Lib):
  - `signal_trend` (Trend Strength) ∈ {−1, 0, +1}
  - `signal_reversion` (Volatility Reversion) ∈ {−1, 0, +1}
  - `signal_breakout` (Volume Breakout) ∈ {−1, 0, +1}

These signals are derived from:

- EMAs (50/200), ADX, MACD
- Bollinger Bands, RSI, MFI
- Bollinger Band width, OBV, ATR

**Output**

The primary model output is a **daily trading position** on the SSE Composite Index:

- Position ∈ {−1, +1}  
  - +1: long index  
  - −1: short index

Intermediate outputs:

- `p_t = P(next-day return > 0)` from logistic regression.
- `strategy_ret_t = position_t * asset_ret_{t+1}` as realised strategy return.

**Model Architecture**

The architecture is a two-layer structure:

1. **Rule-based technical sub-strategies (feature generators)**  
   - Trend Strength strategy: uses EMA(50/200), ADX, and MACD to detect sustained trends.  
   - Volatility Reversion strategy: uses Bollinger Bands, RSI, and MFI to detect mean-reversion in ranging markets.  
   - Volume Breakout strategy: uses Bollinger Band width (squeeze), OBV, and ATR to detect impending breakouts.

   Each sub-strategy outputs a discrete signal in {−1, 0, +1}.

2. **ML meta-model (ensemble combiner)**  
   - Logistic Regression (L2-regularised, `C = 10^logC`) trained on:
     - Features: `[signal_trend, signal_reversion, signal_breakout]`
     - Target: sign of next-day index return (`asset_ret > 0` as 0/1).
   - The model is refit in a cross-validated, walk-forward manner for evaluation and on the full sample for final backtest.

The overall system is thus a **rule-based feature extractor + linear probabilistic classifier** that generates a dynamic trading signal.

---

## Performance

**Metric**

- Primary: **annualised Sortino ratio** of daily strategy returns.
  - Sortino focuses on downside volatility:
    - Numerator: annualised mean excess return.
    - Denominator: square root of annualised downside variance.
- Secondary (for inspection only):
  - Cumulative returns / equity curve.
  - Fold-wise Sortino ratios.

**Evaluation Protocol**

- Data: daily SSE Composite Index (`000001.SS`) from 2010-01-01 to latest available date.
- Cross-validation:
  - Time-series K-fold (e.g. 5 folds) with non-overlapping contiguous segments.
  - For each fold:
    - Training data: all observations strictly before the validation segment.
    - Validation data: the fold segment (future relative to training window).
  - The model is re-trained from scratch on each fold’s training data.
  - Performance metric: Sortino ratio computed on the validation returns only.
- Hyperparameter search:
  - Sobol quasi-random sampling over:
    - `adx_threshold`
    - `bb_reversion_period`
    - `rsi_oversold`
    - `atr_period`
    - `bb_breakout_period`
    - `bb_width_thresh`
    - `logC` (log10 of logistic regression `C`)
  - Objective: maximise **mean Sortino ratio across folds**.

**Summary**

- The model typically yields:
  - Higher cross-validated Sortino ratios than:
    - A naive buy-and-hold on the index.
    - Indicator-level ML models that use raw indicators directly as features.
  - A smoother equity curve with fewer deep drawdowns, driven by:
    - Avoidance of low-ADX, choppy regimes for the trend component.
    - Selective engagement in reversion and breakout conditions when confirmed by volume/momentum.

Exact numerical metrics will depend on the random seed, specific evaluation horizon, and current state of the market data at run time.

---

## Limitations

- **Market specificity**:  
  The model is designed and tuned specifically for the SSE Composite Index. Behaviour on other indices or asset classes is unknown and likely suboptimal without retuning.

- **Stationarity assumption**:  
  The method assumes that relationships between the three sub-strategy signals and future returns are at least partially stable over time. Structural breaks, regime changes, or policy shifts can degrade performance significantly.

- **Limited feature set**:  
  Only three high-level signals are used as features. This is deliberate (to reduce overfitting) but also constrains the information the ML model can exploit, potentially missing important nuance in price/volume dynamics.

- **No transaction costs or slippage in core logic**:  
  The current implementation does not explicitly include commissions, bid–ask spreads, or slippage. Real-world performance would be lower once costs are accounted for.

- **Daily frequency only**:  
  The model operates on daily data. Intraday dynamics, gap risk, and execution timing are abstracted away.

- **Binary direction modelling**:  
  Logistic regression predicts only the sign (positive/negative) of next-day returns, not magnitude. This can misweight days with very small moves versus larger ones.

---

## Trade-offs

- **Bias vs. Flexibility**  
  Using three hand-crafted sub-strategy signals as features injects strong prior structure into the model. This reduces the chance of overfitting but limits flexibility. An indicator-heavy or deep learning model might capture more nuance but would be more fragile and data-hungry.

- **Simplicity vs. Predictive Power**  
  Logistic regression is simple, interpretable, and easy to regularise. It cannot capture complex nonlinear interactions between strategies. Nonlinear models (trees, neural nets) could find richer patterns but would increase variance and reduce transparency.

- **Return vs. Robustness**  
  The objective is cross-validated Sortino, which prioritises robust downside control over maximising raw returns. Some parameter sets might yield higher nominal CAGR but worse drawdowns and lower Sortino; these are intentionally deprioritised.

- **Cross-validation vs. Look-ahead risk**  
  The time-series CV protocol mitigates look-ahead bias by training only on past data in each fold. However, the final “best” model is often refitted on the full dataset for backtesting, which can make the final equity curve appear slightly more optimistic than strictly out-of-sample performance.

- **Generalisability vs. Specialisation**  
  Hyperparameters are tuned for one index and one sampling frequency. Applying the same configuration elsewhere without retraining is a misuse; conversely, fully general models would be less fitted to the microstructure and behaviour of the SSE Composite.

This model card should be read as documentation for a research tool, not a guarantee of real-world profitability or suitability for live trading.