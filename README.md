# BTC Cent Forecast — 4-hour ahead Bitcoin price forecast

Entry for make.fun competition `btc-cent-forecast-01`: read the last 72 hours
of Bitcoin's 1-minute closes (4,320 prices) and forecast the close 240 minutes
after the checkpoint. Node 22, standard library only, no network, deterministic.

## How it works

`main.ts` reads every `*.in` file in `$MAKEFUN_INPUT_DIR` (else `/input`) and
writes `<id>.out` to `$MAKEFUN_OUTPUT_DIR` (else `/output`) as
`{"forecasts":[{"case":"<id>","price":<number>}]}`.

The forecast, in log space, is the checkpoint price nudged **against** its own
last-4-hour move, plus a gentle pull toward its 72-hour mean:

```
drift = -0.15 * (log P_t - log P_{t-240}) - 0.05 * (mean72h(log P) - log P_t)
forecast = exp(log P_t + drift)
```

Why reverse instead of follow the trend? On the practice recording, consecutive
4-hour log returns are slightly *negatively* autocorrelated (lag-1 ≈ -0.31):
a 4-hour move tends to give part of itself back over the next 4 hours. The
weight is deliberately small (-0.15) because the effect is weak and a heavy
reversal bet loses badly on a trending day. The 72-hour mean pull (-0.05) is
small for the same reason. The total nudge is capped at 1.5× the
realised 240-minute volatility (1-minute return std over the last 24 h, scaled
by √240), so a wild 4-hour move can never produce a wild forecast. The result
is rounded to the price grid's 5 decimals and clamped to (0, 10,000,000).

## Backtest (practice cases p00–p04; target of p0k = last price of p0(k+1))

| model | MAE (cents) |
|---|---|
| persistence (house baseline) | 47,054.66 |
| this model | 45,829.90 |

On those five cases — a single strongly down-trending day — the model beats
persistence, though the gap is within noise (paired difference −1,224.76 ±
3,264.95 cents). On 64 hourly checkpoints cut from the same recording (Nov 8–10),
persistence scores 42,870.73 cents vs this model's 41,387.14 (paired
difference −1,483.59 ± 821.38 cents), and the model also beats persistence on the
most recent 24 checkpoints (44,326.66 vs 47,241.58). On 16 non-overlapping
4-hour-spaced checkpoints the model is essentially tied with persistence
(41,900.41 vs 41,768.27). The edge is small and regime-dependent; the model is
a conservative improvement on average, not a bet on any single regime.
Momentum/trend-following variants and smoothed-price variants were tested and
rejected: they scored worse than persistence on the hourly checkpoints.

## Run locally

```
mkdir -p out && MAKEFUN_INPUT_DIR=<kit>/input MAKEFUN_OUTPUT_DIR=out node main.ts
```

One run over all six practice cases takes well under a second; a single case
is far under the 8-second cap. Two runs produce byte-identical outputs.
