# Genesis PROV 9: Bondability -- Claims Summary

**Total Claims:** 13
**Status:** Research (not yet provisional patent filing)
**Validation Level:** Benchmarked against published data; not validated against proprietary fab data

---

## Claim 1: Physics-Based Hybrid Bonding Yield Prediction

**Description:** A computational method for predicting hybrid bonding (Cu-Cu direct bonding) die-level yield from a GDS/OASIS layout input by modeling the complete six-stage physics chain: CMP planarity, contact mechanics, void formation, thermal stress, delamination, and yield.

**Status:** Research. Demonstrated computationally. Validated against published benchmarks (Stine 1998, Turner 2002, Suhir 1986, Murphy 1964). Not validated against proprietary fab data.

---

## Claim 2: CMP Planarity Prediction (Preston Equation Model)

**Description:** A CMP recess prediction method using Effective Planarization Length (EPL) kernel convolution and PCHIP interpolation to map local pad density to surface topography (recess in nanometers), derived from the pressure-velocity dependence in the Preston equation.

**Status:** Research. Default CMP preset is `copper_hybrid_bonding` (Enquist 2019, Kim 2022), with ~3x lower recess than standard Cu damascene. Legacy aluminum calibration (Stine et al. 1998) retained for comparison. Hold-out validation at 6 non-calibration density points. All CMP presets are from published literature, not proprietary fab data -- must be replaced with actual fab measurements for production use.

---

## Claim 3: Contact Mechanics for Cu-Cu Bonding

**Description:** A spectral FFT contact solver that determines bond interface contact state (bridging vs. conforming) by minimizing elastic + adhesion energy in Fourier space, achieving O(N log N) computational complexity per iteration.

**Status:** Research. Validated against Turner & Spearing 2002 qualitative predictions (thick wafer bridges, thin wafer conforms, adhesion reduces gap). Spectral method is standard in tribology literature; contribution is application to hybrid bonding yield pipeline.

---

## Claim 4: Anneal Diffusion Model for Bond Strength

**Description:** A thermal stress analysis method that computes CTE-mismatch-driven stress during post-bond annealing using Voigt/Reuss mixing rules for composite modulus and plane-stress FEA, with interface fracture mechanics for delamination risk.

**Status:** Research. Benchmarked against Suhir 1986 analytical bimetallic strip solution. Linear elastic only (no plasticity). Conservative bias at sharp interfaces.

---

## Claim 5: Monte Carlo Uncertainty Quantification

**Description:** A Monte Carlo yield prediction engine that samples process parameter distributions (overlay sigma, particle density, surface roughness) and propagates uncertainty through the full physics chain to produce yield distributions (P10/P50/P90) with sensitivity analysis.

**Status:** Research. Sensitivity analysis identifies correlation_length_um as dominant parameter (yield swing of 40-99% across plausible range; see HONEST_DISCLOSURES.md #7). 200 MC samples by default (below convergence minimum of 1,000; see HONEST_DISCLOSURES.md #14). Validated for monotonicity, bounds, and seed sensitivity.

---

## Claim 6: Design Rule Compiler for Bond-Aware Layout

**Description:** A DRC rule compilation system that translates physics solver outputs (CMP recess, void risk, delamination risk, thermal stress) into layout-level design rule violations with spatial violation masks and KLayout marker database output.

**Status:** Research. Generates actionable violation masks. Threshold-based heuristic detection. Does not detect specific layout structures (guard rings, seal rings).

---

## Claim 7: Inverse Design Reliability Compiler

**Description:** A multi-objective inverse design optimizer that jointly minimizes bond void risk, delamination risk, CMP non-uniformity, and thermal stress by modifying fill density patterns, using both additive (hill-climbing) and subtractive (stress-minimizing) strategies with multi-start restarts.

**Status:** Research. Demonstrated yield improvement on synthetic layouts. Gradient-free optimization. No gradient-based (adjoint) method.

---

## Claim 8: Gradient-Compensated Dummy Fill

**Description:** A dummy fill algorithm that compensates for density gradient effects on CMP planarity by placing fill patterns that minimize both CMP recess variation and downstream bonding failure risk, rather than optimizing CMP uniformity alone.

**Status:** Research. Additive hill-climbing strategy targets highest combined-risk tiles. Multi-objective fitness function includes void risk, delamination risk, CMP sigma, and thermal stress.

---

## Claim 9: Stress-Resonant Spacing Avoidance

**Description:** A design rule derived from parametric study showing that periodic density patterns at spacings matching the stress diffusion length (~10 um for Cu/SiO2) create resonance-like thermal stress amplification. The rule identifies and avoids these stress-resonant spacings.

**Status:** Research. Observed in computational parameter sweeps. Subtractive optimizer specifically targets stress hotspots from this effect. Not validated experimentally.

---

## Claim 10: Discrete Inverse Design via Sigmoid Projection

**Description:** A method for optimizing discrete (binary) fill patterns using continuous relaxation with sigmoid projection, enabling gradient-compatible optimization of inherently discrete layout decisions.

**Status:** Research. Archived in development history. Experimental technique.

---

## Claim 11: Evolutionary Inverse Design with Genetic Algorithm

**Description:** A genetic algorithm approach to fill pattern optimization that explores the combinatorial design space of discrete fill placements, complementing the gradient-based and hill-climbing optimization strategies.

**Status:** Research. Archived in development history. Experimental technique.

---

## Claim 12: Spectral FFT Solver with Dugdale Cohesive Zone

**Description:** The specific combination of spectral (Fourier-domain) elastic kernel with Dugdale cohesive zone adhesion model, using smoothed piecewise-linear potential (Huber-like softplus, alpha=20) for numerical stability, solved via L-BFGS-B with box constraints.

**Status:** Research. Individual components (spectral methods, Dugdale CZM, L-BFGS-B) are established techniques. The specific integration for hybrid bonding contact mechanics within a yield prediction pipeline is the contribution.

---

## Claim 13: End-to-End GDS-to-Yield Pipeline

**Description:** A complete software pipeline that accepts GDS/OASIS semiconductor layout files, extracts pad density features, runs the six-stage physics chain, and produces yield predictions, risk maps, DRC violations, sensitivity analysis, and HTML signoff reports.

**Status:** Research. 61 production Python source files, 17 test files (all passing), CLI with 7 commands, REST API with 5 endpoints. Architecturally complete. Not deployed in production.

---

## Summary Table

| # | Claim | Key Technique | Validation |
|---|-------|---------------|------------|
| 1 | Physics-based yield prediction | 6-stage physics chain | Published benchmarks |
| 2 | CMP planarity | Preston/PCHIP/EPL | Stine 1998 (hold-out) |
| 3 | Contact mechanics | Spectral FFT + Dugdale | Turner 2002 |
| 4 | Anneal thermal stress | Voigt mixing + FEA | Suhir 1986 |
| 5 | Monte Carlo UQ | Murphy/Stapper + MC | Sanity checks |
| 6 | Design rule compiler | Threshold-based DRC | Functional tests |
| 7 | Inverse design optimizer | Multi-objective fill | Synthetic layouts |
| 8 | Gradient-compensated fill | Joint CMP+bonding fill | Synthetic layouts |
| 9 | Stress-resonant spacing | Parametric sweep rule | Computational only |
| 10 | Discrete sigmoid projection | Continuous relaxation | Experimental |
| 11 | Evolutionary GA design | Genetic algorithm | Experimental |
| 12 | Spectral + Dugdale CZM | FFT + cohesive zone | Turner 2002 |
| 13 | GDS-to-yield pipeline | End-to-end platform | Integration tests |

---

## Appendix: correlation_length_um Calibration Guide

### What is correlation_length_um?

`correlation_length_um` (default: 500 um) controls the spatial autocorrelation of bonding defects in the Monte Carlo yield model. It determines how many *independent failure clusters* exist on a die:

    n_clusters = (die_H * die_W) / (correlation_length / tile_um)^2

For example, on a 64x64-tile die with 25 um tiles and correlation_length = 500 um (= 20 tiles), n_clusters ~ 10. Each cluster fails or succeeds semi-independently; total yield is the product of per-cluster survival probabilities.

### Why it dominates yield by ~100x

This single parameter swings yield predictions by roughly 100x more than all other sampled parameters (overlay sigma, particle density, roughness) combined:

- correlation_length = 100 um --> many independent clusters --> yield ~ 40%
- correlation_length = 500 um (default) --> moderate clustering --> yield ~ 85%
- correlation_length = 1000 um --> few clusters --> yield ~ 99%

The physics: shorter correlation lengths mean defects are spatially uncorrelated, so each small region of the die independently rolls the dice on bonding success. Longer correlation lengths mean defects clump together, and most of the die bonds successfully while a few localized regions contain all the failures -- overall yield is much higher because one bad cluster does not kill the whole die.

**Yield predictions are only as reliable as this calibration.** All other model improvements (CMP fidelity, contact solver accuracy, thermal stress) are secondary to getting this parameter right.

### How to calibrate (fab partner workflow)

In order of decreasing reliability:

1. **BEST -- Wafer inspection autocorrelation.** Obtain a bonding defect map from KLA/Onto SAM (Scanning Acoustic Microscopy) or IR transmission imaging of bonded wafer pairs. Compute the 2D spatial autocorrelation of the binary defect map. The distance at which the autocorrelation drops to 1/e is the correlation_length. Typical result for Cu-Cu hybrid bonding: 200-800 um.

2. **GOOD -- Layout autocorrelation proxy.** Use `estimate_correlation_length()` on a real density map extracted from production GDS. This uses the layout density autocorrelation as a proxy for defect correlation. Less accurate than direct defect measurement but requires no fab data.

3. **FAIR -- Published literature values.** Cunningham 1990 reports 200-500 um for generic IC defects. Turner & Spearing 2002 reports 100-300 um for direct bonding defects. Use these as starting points and bracket with sensitivity analysis.

4. **WORST -- Use the default (500 um).** This is an educated guess within the plausible range. Always run sensitivity analysis (`sensitivity_analysis: true` in config) to understand how much yield predictions change across the 100-1000 um range.

### Sensitivity range

| correlation_length_um | Approximate yield (typical die) | Confidence |
|-----------------------|--------------------------------|------------|
| 100 um | ~30-50% | Too pessimistic for most processes |
| 200 um | ~50-70% | Conservative (early process) |
| 500 um (default) | ~80-90% | Moderate (mature process) |
| 1000 um | ~95-99% | Optimistic (best-in-class) |
| 2000 um | ~99%+ | Entire die acts as one cluster |

The realistic range is 100-2000 um. Values below 100 um produce unrealistically many independent clusters; values above 2000 um make the entire die a single cluster (too optimistic).

### Configuration

In `process_example.yaml` or programmatically:

```yaml
yield_model:
  correlation_length_um: 500.0   # MUST calibrate to your process
  sensitivity_analysis: true     # Always run sensitivity sweeps
```

See also: `bondability/config.py` (YieldModelConfig), `HONEST_DISCLOSURES.md` #7.

---

*All claims are research-status. No provisional patent has been filed. Validation is against published academic data, not proprietary fab data.*
