# Air passengers: forecast recommendation

**Series:** monthly international air passengers, Jan 1949 – Dec 1960 (144 months, no gaps).
**Task:** forecast the next 12 months; recommend a model, backed by an 8-window rolling-origin evaluation (h = 12 per window).

## The recommendation

**Ship AutoETS(M,N,M) — multiplicative error, no trend, multiplicative seasonality.** It beats the seasonal-naive floor on every metric the cross-validation harness reports: **MASE 0.95 vs. 1.31**, a ~27% smaller error than the naive "same month last year" benchmark, on the same 8 rolling folds. That one number is the headline, but it isn't the whole case — the win holds on the full predicted distribution too (scaled CRPS 0.050 vs. 0.062, RMSSE 1.00 vs. 1.25), not just the point forecast. In this series the seasonal pattern grows with the level of travel rather than staying a fixed size, and ETS's multiplicative seasonal term captures that directly; a plain "repeat last year" rule cannot.

| Model | MASE | RMSSE | CRPS | 80% coverage |
|---|---|---|---|---|
| Seasonal naive (floor) | 1.31 | 1.25 | 0.062 | 51% |
| **AutoETS (recommended)** | **0.95** | **1.00** | **0.050** | 49% |

*(All four numbers come from the same 8-window rolling-origin harness — h=12, step=12 — not a single holdout.)*

## The intervals

Neither model's 80% interval is trustworthy as stated, and that matters as much as the win above. Coverage is the check a nice-looking band cannot do by itself: with only 8 folds, seasonal naive's interval covers the true value 51% of the time and AutoETS's covers 49% — both far short of the 80% they're labeled with. In other words, both models' stated bands are **too narrow** for how much the forecast actually moves from fold to fold (AutoETS's fold-level MASE ranges from 0.56 in the easiest fold to 1.55 in the hardest — a 3x spread across only 8 windows). The 24-month forecast fan widens from about ±28 passengers (thousands) at one month out to about ±62 at two years out, which looks reasonable on the chart — but the coverage check says the true uncertainty is wider than that. **Treat the point forecast as the deliverable and the stated interval as optimistic**, not as a calibrated risk band, until it's re-checked on more folds or built from resampled residuals rather than the model's own formula.

## The residuals

The seasonal-naive floor's residuals fail the Ljung-Box test decisively at both 12 and 24 lags (p ≈ 2×10⁻⁴³ and 2×10⁻⁴⁴) — nowhere close to white noise. That's expected and diagnostic: "repeat last year" throws away the trend entirely, so every residual still carries a slice of year-over-year growth, plus a bit of amplitude mismatch because the seasonal swing itself is growing. That is precisely the structure an explicit, adaptively-estimated level and a multiplicative seasonal term is built to absorb, which is why ETS closes most of that gap. What ETS itself still misses is the *trend* — the search picked no explicit trend component (the "N" in M,N,M), relying instead on an adaptive level (smoothing weight ≈ 0.37) to track growth implicitly. That's a reasonable in-sample compromise but a real risk looking forward: if passenger growth keeps accelerating rather than continuing at its recent pace, a trend-free level will under-forecast, and that risk doesn't show up anywhere in the reported MASE — it would only appear in future folds we haven't seen yet.

## One change

**Re-run the cross-validation with more, shorter windows (e.g., 16 windows at h=6 instead of 8 windows at h=12).** The current 80%-interval coverage numbers (51%, 49%) are built from only 8 scored windows, which is too few to tell a genuinely miscalibrated band from ordinary sampling noise — doubling the number of folds directly tightens that estimate without touching the series or the model. I'd expect this to either confirm the interval is too narrow (in which case the fix is a resampled-residual interval, not a bigger AutoETS search) or reveal that 49-51% coverage was itself partly a small-sample artifact and the true long-run coverage is closer to 65-70%. Either answer is actionable; the current sample size is not enough to act on with confidence.

---
*Every number above comes out of the notebook's rolling-origin harness (`coursekit.scoring.score_cv`); none is from a single train/test split.*
