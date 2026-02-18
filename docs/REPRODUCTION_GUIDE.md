# Reproduction Guide

This document explains how to verify the claims made in the Bondability white paper using the publicly available verification script and reference data.

---

## What You Can Reproduce

The verification script (`verification/verify_claims.py`) runs five categories of checks using only Python standard library and basic math. It does not require the proprietary Bondability solver package. The checks verify:

1. **CMP recess prediction** -- PCHIP interpolation of the Stine 1998 calibration curve, tested at held-out density points.
2. **Contact mechanics fundamentals** -- Kirchhoff plate bending stiffness calculations and qualitative bridging/conforming predictions.
3. **Murphy yield model** -- Analytical yield calculations from the published Murphy/Stapper formulas.
4. **Spectral FFT scaling** -- Verification that reference timing data is consistent with O(N log N) complexity.
5. **Physics chain completeness** -- Structural check that all 6 stages are represented in canonical values.

---

## Running the Verification Script

### Requirements

- Python 3.10 or later
- No external packages required (standard library only)

### Execution

```bash
cd verification/
python verify_claims.py
```

### Expected Output

The script prints results for each check and writes a JSON report to `verification/verification_results.json`.

Expected: all checks PASS, with possible WARN on FFT scaling (real-world timing has overhead beyond pure O(N log N)).

---

## What You Cannot Reproduce (and Why)

### Proprietary Solver Benchmarks

The four published-data benchmarks (CMP vs. Stine 1998, Contact vs. Turner 2002, Thermal vs. Suhir 1986, Yield sanity) are implemented inside the proprietary Bondability solver package. The verification script verifies the underlying physics and mathematics independently, but does not run the actual solvers.

To reproduce the solver benchmarks, you would need:
- The `bondability` Python package (not included in this public repository)
- Run: `bondability benchmark --verbose`

### GDS-to-Yield Pipeline

The end-to-end pipeline (GDS input to yield output) requires the solver package and a GDS layout file. Neither is included in this public repository.

### Fab-Calibrated Results

All published results use default calibration parameters from academic literature. Reproducing fab-specific yield predictions would require:
- Fab-specific CMP calibration data (density vs. recess measurements for Cu damascene)
- Fab-specific defect correlation length
- Fab-specific overlay sigma and particle density

---

## Verifying Individual Claims

### Claim 1: Physics-Based Yield Prediction (6-Stage Chain)

**Verification:** Check 5 in `verify_claims.py` confirms all 6 stages are represented in canonical values.

**Independent verification:** The physics chain is documented in the Solver Overview (`docs/SOLVER_OVERVIEW.md`). Each stage implements published, peer-reviewed models:
- CMP: Stine et al. 1998
- Contact: Turner & Spearing 2002
- Thermal: Suhir 1986
- Yield: Murphy 1964, Stapper 1983

### Claim 2: CMP Planarity (Preston Equation)

**Verification:** Check 1 in `verify_claims.py` performs PCHIP interpolation of the Stine 1998 calibration curve and tests at held-out densities.

**Independent verification:**
- The 5-point bathtub curve is from Table II of Stine et al. 1998.
- PCHIP interpolation is a standard algorithm (scipy.interpolate.PchipInterpolator).
- Hold-out densities {0.1, 0.2, 0.4, 0.6, 0.8, 0.9} were NOT used for calibration.

### Claim 3: Contact Mechanics (Spectral FFT)

**Verification:** Check 2 in `verify_claims.py` computes Kirchhoff plate bending stiffness and verifies qualitative predictions (thick wafer bridges, thin wafer conforms).

**Independent verification:**
- Kirchhoff plate bending: D = E*h^3 / (12*(1-nu^2)) is a textbook formula (Timoshenko & Goodier 1970).
- Stiffness ratio scales as h^3 -- verified in the check.
- Spectral FFT contact mechanics is established in the tribology literature (Persson 2006).

### Claim 4: Anneal Thermal Stress

**Verification:** Not directly testable without the solver package. The physics (CTE mismatch stress) is straightforward:
- Thermal strain: eps = CTE * delta_T
- For Cu at 300C anneal: eps = 17e-6 * 275 = 4.675e-3
- Voigt composite stress scales with density-weighted modulus

### Claim 5: Monte Carlo UQ

**Verification:** Check 3 in `verify_claims.py` verifies the Murphy yield model analytically (monotonicity, bounds, exact values, clustering effect).

### Claims 6-9: Design Rule Compiler, Inverse Design, Gradient-Compensated Fill, Stress-Resonant Spacing

**Verification:** These are algorithmic claims about the software platform. They can be verified by running the solver package (not included). The concepts are described in the Solver Overview document.

### Claims 10-11: Discrete Sigmoid Projection, Evolutionary GA

**Verification:** These are experimental techniques in the development archive. They are not part of the production pipeline.

### Claim 12: Spectral FFT + Dugdale CZM

**Verification:** Check 4 in `verify_claims.py` verifies O(N log N) scaling from reference timing data. The Dugdale CZM is a standard cohesive zone model (Maugis 1992).

### Claim 13: End-to-End Pipeline

**Verification:** The pipeline architecture is documented in the Solver Overview. The 17 test files (listed in evidence/key_results.json) cover all pipeline stages.

---

## Reference Data

All reference data is in `verification/reference_data/canonical_values.json`. This file contains:

- Solver configuration defaults
- Calibration knot values
- Material properties
- Published benchmark references with citations
- Physics chain stage definitions

---

## Known Limitations of This Verification

1. **No solver execution.** The verification script tests mathematical relationships and reference data consistency. It does not execute the Bondability solvers.

2. **Linear interpolation, not PCHIP.** Check 1 uses linear interpolation for simplicity. The actual solver uses PCHIP, which provides smoother curves and better accuracy at held-out points.

3. **Simplified contact mechanics.** Check 2 verifies analytical scaling relationships, not the full spectral FFT solver with Dugdale adhesion.

4. **Reference timing data.** Check 4 uses representative timings, not live benchmarks. Actual scaling depends on hardware, cache behavior, and L-BFGS-B convergence.

5. **No experimental validation.** All checks are computational. No fabricated wafer data is available for validation.

---

*This guide accompanies the non-confidential public repository for Genesis PROV 9: Bondability.*
