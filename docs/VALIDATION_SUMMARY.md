# Validation Summary: What's Verified and What's Not

> Radical transparency about the boundaries of our validation.

---

## Validated Envelope

Genesis Bondability has been computationally validated for the following conditions:

| Parameter | Validated Range | Notes |
|-----------|----------------|-------|
| Bonding metallurgy | Cu-Cu | Primary target; other metallurgies need recalibration |
| Pad pitch | <=25um tile resolution | Default grid; finer with mesh refinement |
| Anneal temperature | 200-400C | Linear elastic regime |
| Dielectric | SiO2/Cu stacks | Standard hybrid bonding stack |
| Die dimensions | Up to 30x30mm | 64x64 default grid at 25um tiles |
| Overlay sigma | 1-50nm | Full range of current bonding tools |
| Surface roughness | 0.1-5.0nm RMS | Post-CMP surface quality range |

## NOT Validated

The following fall **outside** the current validation envelope:

| Condition | Status | What's Needed |
|-----------|--------|---------------|
| Plasticity regime | Not modeled | Incremental plasticity solver (load stepping, radial return) |
| Sub-5nm features | Not tested | Mesh refinement validation at finer pitch |
| 3D multi-layer stacks | Not modeled | Inter-layer CMP interaction model |
| Non-Cu metallurgy (Al, W, Co) | Not calibrated | Material property library + fab data |
| Wafer-level warpage | Input only | First-principles warpage solver |
| Creep / viscoelastic effects | Not modeled | Time-dependent material models |
| Production fab data | None collected | Fab partner calibration (10+ wafers) |

## Published-Data Benchmarks

All solvers are benchmarked against published academic data:

| Benchmark | Reference | What's Tested | Result |
|-----------|-----------|--------------|--------|
| CMP recess prediction | Stine et al. 1998, IEEE TSM | Hold-out validation at 6 non-calibration densities | PASS |
| Contact mechanics | Turner & Spearing 2002, J. Appl. Phys. | Thick bridges, thin conforms, adhesion reduces gap | PASS |
| Thermal stress | Suhir 1986, J. Appl. Mech. | FEA vs. analytical bimetallic strip solution | PASS |
| Yield model | Murphy 1964, Proc. IEEE | Monotonicity, bounds, seed sensitivity | PASS |

**Important caveat:** These benchmarks verify that the solvers implement the published
physics correctly. They do **not** verify that the models predict the yield of a specific
fab process. That requires fab-specific calibration data.

## Key Limitations That Affect Predictions

1. **correlation_length dominates yield** (Sobol S1 = 0.582). This single parameter
   swings yield predictions by 20+ percentage points. It must be calibrated per-fab
   using wafer inspection data or KLA Archer overlay autocorrelation. Default value
   (500um) is an educated guess.

2. **FNO surrogate R-squared = 0.50.** The neural surrogate is screening quality only --
   use for fast ranking of layout candidates, not for quantitative yield prediction.
   Always validate FNO results with the full physics pipeline.

3. **CMP calibration is from literature, not fab data.** Default preset uses published
   Cu hybrid bonding parameters (Enquist 2019, Kim 2022). Production deployment requires
   replacing with fab-specific CMP measurements.

4. **Contact solver is approximate at tile scale.** The Dugdale cohesive zone (dc = 5nm)
   operates at nanometer scale, but the default tile resolution is 25um. Quantitative
   gap values carry 2-5x error. Use mesh refinement for quantitative accuracy.

5. **Monte Carlo default (200 samples) is below convergence.** Increase to 1,000+ for
   quantitative yield predictions. Default is set for demo speed.

## Verification

Run the standalone verification script to reproduce all benchmarks:

```bash
pip install -r verification/requirements.txt
python verification/verify_claims.py
```

Requires Python 3.10+. No proprietary packages. No GPU. No fab data.

See [REPRODUCTION_GUIDE.md](REPRODUCTION_GUIDE.md) for detailed instructions.

## Contact

**Nicholas Harris** -- [nick@nmk.ai](mailto:nick@nmk.ai)

---

*This document is part of the Genesis PROV 9: Bondability public data room.
See [HONEST_DISCLOSURES.md](../HONEST_DISCLOSURES.md) for the complete list of 15 documented limitations.*
