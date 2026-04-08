# Deployment Checklist

> What you need to evaluate and deploy Genesis Bondability.

---

## Data Format

### Input

Genesis accepts layout and process parameters as JSON or CSV:

| Parameter | Type | Range | Description |
|-----------|------|-------|-------------|
| die_width_mm | float | 1-30 | Die width in millimeters |
| die_height_mm | float | 1-30 | Die height in millimeters |
| pad_density | float | 0.01-1.0 | Local Cu pad density (area fraction) |
| overlay_sigma_nm | float | 1-50 | Overlay alignment sigma (nm) |
| anneal_temp_C | float | 200-400 | Post-bond anneal temperature |
| roughness_rms_nm | float | 0.1-5.0 | Surface roughness RMS (nm) |
| oxide_thickness_nm | float | 10-500 | Dielectric thickness (nm) |

For GDS/OASIS input, the pipeline extracts pad density automatically.

### Output

JSON response containing:

- **yield_p50**: Median yield prediction (0-1)
- **yield_p10 / yield_p90**: Confidence interval
- **risk_heatmap**: Per-tile bonding failure probability (2D array)
- **dominant_sensitivity**: Top 3 parameters driving yield variance
- **drc_violations**: Layout rule violations with spatial coordinates
- **verdict**: GO / MARGINAL / NO-GO

## API Authentication

All API endpoints require an `X-API-Key` header.

| Tier | Rate Limit | Features |
|------|-----------|----------|
| Free | 10 layouts/day | Single layout, no batch |
| Starter | 1,000 layouts/month | Batch processing, JSON export |
| Professional | Unlimited | Priority queue, risk heatmaps, DOE campaigns |
| Enterprise | Custom | On-premise, custom calibration, source access |

## Validation Protocol

To verify predictions on your first layouts:

1. **Start with synthetic layouts.** Run 3 known configurations (low/medium/high density)
   through the pipeline. Confirm monotonic yield behavior (higher density = more contact = higher yield, up to CMP limits).

2. **Compare against verify_claims.py.** Run the standalone verification script to confirm
   your installation reproduces the published benchmarks.

3. **Run a 10-layout DOE.** Vary overlay sigma and anneal temperature across your expected
   process window. Confirm the yield response surface is physically reasonable.

4. **Calibrate with 10 wafers.** If you have experimental yield data, run Bayesian
   calibration to converge correlation_length. Expect CI < 20um after 10 wafers.

## Performance

| Operation | Latency | Notes |
|-----------|---------|-------|
| Full physics pipeline (1 layout) | <1 second | 7-stage chain, CPU |
| FNO neural surrogate (1 die) | 13 ms | CPU inference, screening quality |
| API end-to-end (1 layout) | <100 ms | Excludes GDS parsing |
| 10K DOE campaign | ~3 hours | Single CPU, parallelizable |
| Bayesian calibration (10 wafers) | ~5 minutes | Sequential update |

## Infrastructure Requirements

| Requirement | Specification |
|-------------|--------------|
| Runtime | Python 3.10+ |
| Memory | 4 GB RAM minimum |
| GPU | Not required (CPU inference for FNO) |
| Storage | <1 GB for application + models |
| Container | Docker image available (single port 8000) |
| Database | SQLite (embedded, no external DB) |
| Monitoring | Prometheus metrics endpoint included |
| OS | Linux, macOS, Windows (WSL) |

## What You Need

- Python 3.10 or later
- 4 GB RAM (8 GB recommended for batch processing)
- No GPU required -- all inference runs on CPU
- Network access for API mode (or run fully offline for on-premise)

## What You Get

- Yield P50/P90 with confidence intervals
- Per-die risk heatmap (spatial failure probability)
- Dominant sensitivity parameters (which process knobs matter most)
- DRC violation list with spatial coordinates
- GO / MARGINAL / NO-GO verdict
- HTML signoff report (for engineering review)

## Contact

**Nicholas Harris** -- [nick@nmk.ai](mailto:nick@nmk.ai)

---

*This document is part of the Genesis PROV 9: Bondability public data room.
See [HONEST_DISCLOSURES.md](../HONEST_DISCLOSURES.md) for limitations and caveats.*
