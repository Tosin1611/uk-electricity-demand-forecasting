# uk-electricity-demand-forecasting

Short-term forecasting of Great Britain electricity demand using real, public grid data from **NESO** (National Energy System Operator), the body that runs the GB electricity grid.

## Headline result

| Model | MAE (MW) | MAPE |
|---|---|---|
| Naive seasonal baseline (same half-hour, 7 days prior) | 1,755 | 8.08% |
| Prophet (calendar-only, no weather input) | **1,722** | **7.81%** |

A modest ~2% error reduction over the naive baseline on a 14-day out-of-sample test — a small, honest result rather than an inflated one.

## What this project does

1. Pulls ~11,950 half-hourly **National Demand (ND)** readings for 2026 directly from NESO's open [Data Portal API](https://www.neso.energy/data-portal/historic-demand-data).
2. Builds a clean, timestamp-indexed time series and validates it (checks for duplicates, gaps, and non-actual/forecast rows).
3. Builds a naive seasonal baseline: predict each half-hour using the same half-hour exactly one week earlier.
4. Fits a [Prophet](https://facebook.github.io/prophet/) time-series model and evaluates both on a 14-day holdout, scored by MAE and MAPE.

## The bug I caught

The first Prophet fit, with yearly seasonality enabled, scored **33,837 MW MAE / 155% MAPE** — worse than useless. Plotting forecast against actual showed the model's trend line running straight downward through the whole test window, eventually predicting negative demand.

**Cause:** training data only covered January–August 2026 (a real seasonal decline from winter into summer), so the model had never seen an autumn and simply extrapolated the decline forward — exactly what Prophet's own startup warning had flagged (*"yearly seasonality enabled with less than 730 days of history"*).

**Fix:** disabled yearly seasonality, since under a year of data can't support fitting it reliably. That single change brought MAPE down from 155% to 7.81%.

## The real finding: neither model wins everywhere

Breaking the error down by hour of day showed a clean crossover, not a uniform winner:

- **Overnight (00:00–06:00):** the naive baseline wins clearly. Demand is flat and repeats closely week to week, so copying last week's value beats a smoothed average curve.
- **Midday (10:00–16:00):** Prophet wins clearly. This is when embedded solar output is highest and most weather-dependent — error correlation with embedded solar generation was **0.57** for the naive baseline, dropping to **0.28** for Prophet.

Neither model uses solar or weather data as an input — using same-day realised solar output would be look-ahead leakage, since a real forecast wouldn't know that yet. Prophet's improvement here comes only from learning the *average* seasonal solar-suppression pattern, which is more robust than blindly copying last week's noise, but it can't correct for a specific day being sunnier or cloudier than usual.

## Limitations & next steps

- **No weather/solar forecast input** — the remaining midday error is largely weather-driven; closing it further needs an actual forecast feed, not more tuning.
- **Under a year of training history** — yearly seasonality is unreliable below ~2 years of data.
- **Single train/test split** — a rolling/walk-forward backtest across several holdout windows would be more robust than one 14-day split.
- **No hybrid model** — given the clean overnight-vs-midday crossover, a model that blends naive and Prophet by time of day would likely beat both outright.

## Data source

[NESO Historic Demand Data](https://www.neso.energy/data-portal/historic-demand-data) — public, no authentication required. `ND` (National Demand) is the target variable; see the notebook for the full field reference.

## Tools

Python · pandas · Prophet · Google Colab
