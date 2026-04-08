# KLA Archer Integration Brief: Overlay Metrology to Yield Prediction

> Every KLA Archer installation becomes a yield prediction node.

---

## The Bridge

KLA Archer measures post-bond overlay (dx, dy) across the wafer with sub-nanometer
precision. Today, that data goes into SPC charts and disposition decisions. It does
not predict yield.

Genesis Bondability bridges that gap. Archer overlay CSV goes in. Yield prediction
comes out in under 100 milliseconds. The metrology data KLA already collects becomes
the input to a physics-based yield model -- no additional measurement steps, no new
hardware, no changes to the Archer workflow.

## The Result

- **10 wafers** to converge correlation_length to CI < 20um (vs. 100+ wafers manual)
- **Bayesian calibration** uses Archer overlay as the observation model
- **Per-die yield prediction** with P10/P50/P90 confidence intervals
- **Risk heatmap** showing spatial distribution of bonding failure probability

## How It Works

```
KLA Archer CSV (dx, dy per site)
        |
        v
Spatial Autocorrelation Analysis
  -- compute 2D autocorrelation of overlay field
  -- extract correlation_length (1/e decay distance)
        |
        v
Genesis Bayesian Yield Update
  -- prior: physics-based yield from layout + process params
  -- likelihood: overlay correlation vs. defect correlation model
  -- posterior: calibrated yield prediction
        |
        v
Output: Yield P50, Risk Map, GO/NO-GO Verdict
  -- feeds back to Archer workflow as disposition input
```

No Genesis source code is exposed. The integration is data-in, prediction-out.

## Input Format

Genesis accepts the standard Archer export format:

| Field | Type | Description |
|-------|------|-------------|
| site_x | float | Measurement site X coordinate (mm) |
| site_y | float | Measurement site Y coordinate (mm) |
| dx | float | Overlay error X (nm) |
| dy | float | Overlay error Y (nm) |

Additional context (wafer ID, lot ID, process step) is optional metadata.

## Output Format

JSON response containing:

| Field | Type | Description |
|-------|------|-------------|
| yield_p50 | float | Median yield prediction (0-1) |
| yield_p10 | float | 10th percentile yield (pessimistic) |
| yield_p90 | float | 90th percentile yield (optimistic) |
| correlation_length_um | float | Estimated defect correlation length |
| risk_map | array | Per-die bonding failure probability |
| verdict | string | GO / MARGINAL / NO-GO |
| confidence | string | Calibration confidence level |

## Why KLA Benefits

KLA's metrology platform is the industry standard for overlay measurement.
Genesis makes that measurement *more valuable* by connecting it to yield prediction.

- **Existing Archer installations** gain yield prediction capability with zero hardware changes
- **Data already collected** becomes the input -- no new measurement recipes needed
- **Faster process qualification** -- 10 wafers to calibrated yield instead of 100+
- **Stickier customer relationship** -- yield prediction creates a reason to measure more often

The value proposition is not competitive with KLA. It is complementary. Genesis adds
a prediction layer on top of metrology data that KLA already owns.

## Demonstrated Scenarios

Genesis has been validated on three overlay scenarios:

| Scenario | Overlay Sigma | Predicted Yield | Verdict |
|----------|--------------|-----------------|---------|
| Good wafer | 2 nm | >90% | GO |
| Marginal wafer | 8 nm | 60-80% | MARGINAL |
| Bad wafer | 15 nm | <40% | NO-GO |

Additionally, a 10,000-point design of experiments campaign demonstrates process window
mapping across the full overlay-temperature-roughness parameter space, with 85.9% mean
yield across the explored space.

## Contact

**Nicholas Harris** -- [nick@nmk.ai](mailto:nick@nmk.ai)

We are seeking a pilot integration with a KLA Archer installation at a foundry
or OSAT partner. The integration requires only CSV export access -- no modifications
to the Archer platform.

---

*This document is part of the Genesis PROV 9: Bondability public data room.
See [HONEST_DISCLOSURES.md](../HONEST_DISCLOSURES.md) for limitations and caveats.*
