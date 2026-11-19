# GTA6 Trophy Rarity Forecast — make.fun Entry

This is a single-file TypeScript program (`main.ts`) that forecasts trophy rarity percentages for Grand Theft Auto VI at a target time, given historical rarity reads from Exophase.

## Model Summary

1. **Log–log trends**: For each trophy, convert each historical read to hours since the game's release (00:00 UTC on `game.release_date`). Clamp rarity to `[0.01, 99.99]`, then build a series of distinct points `(x, y) = (ln(hours), ln(rarity/100))`.

2. **Per-trophy slopes**: Fit an ordinary least-squares slope of `y` on `x` for trophies with ≥3 distinct points.

3. **Pooled slope**: Take the median of all per-trophy slopes, clamped to `[0, 2.5]`.

4. **Shrinkage**: Blend each trophy's own slope toward the pooled slope with weight `w = min(0.75, n/6)` where `n` is the number of distinct points. Clamp the blended slope to `[0, 2.6]`. If no trophy yielded a slope (flat history), keep the last rarity unchanged.

5. **Projection**: Project the log-rarity forward using a tanh-capped step:
   ```
   y_target = y_last + CAP * tanh(slope * (ln(h_target) - ln(h_last)) / CAP)
   ```
   with `CAP = 3.0`. Median rarity = `exp(y_target) * 100`. If the last rarity ≥ 99.5, forecast 100 for all quantiles.

6. **Quantiles via logistic spread**: Base sigma = `0.6 + 0.25 * ln(1 + horizon_days)`. Add a launch-night refresh term of `0.4` (in quadrature) when the case cutoff is within 24 hours of release and no trophy produced a slope. Clamp total sigma to `[0.45, 1.30]`. Quantiles are `100 / (1 + exp(-(y_target + z * sigma)))` for `z ∈ {−1.645, −0.674, 0, 0.674, 1.645}` (levels 0.05, 0.25, 0.5, 0.75, 0.95). Round to 4 decimals and enforce non-decreasing order per row.

## Running

```bash
# Smoke test with provided samples
mkdir -p out-smoke
MAKEFUN_INPUT_DIR=dev/sample MAKEFUN_OUTPUT_DIR=out-smoke node --experimental-strip-types main.ts

# Production (grader sets these env vars)
MAKEFUN_INPUT_DIR=/input MAKEFUN_OUTPUT_DIR=/output node --experimental-strip-types main.ts
```

## Requirements

- Node 22 (`node --experimental-strip-types`)
- Standard library only, no dependencies, no network access
- Output is deterministic byte-for-byte across runs

## Output Format

Each `<id>.out` file contains:
```json
{
  "trophies": [
    { "id": 1, "q": [q05, q25, q50, q75, q95] },
    ...
  ]
}
```
One row per trophy (55 total), each id exactly once, five non-decreasing finite numbers in `[0, 100]` rounded to 4 decimals.