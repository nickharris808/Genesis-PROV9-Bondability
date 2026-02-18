# Solver Architecture Overview

This document describes the solver architecture of the Bondability platform at a conceptual level. No source code is included. The purpose is to explain the physics and algorithms behind each stage of the pipeline.

---

## Pipeline Architecture

Bondability operates as a sequential six-stage pipeline. Each stage consumes outputs from prior stages and produces typed results for downstream stages. The pipeline accepts GDS/OASIS layout files (or pre-computed density arrays) as input and produces yield predictions, risk maps, and design rule violations as output.

```
Input: GDS/OASIS layout or density array (.npy)
  |
  |-- Stage 1: Feature Extraction
  |-- Stage 2: CMP Recess Prediction
  |-- Stage 3: Bond Void Risk (Contact Mechanics)
  |-- Stage 4: Anneal Delamination Risk (Thermal Stress)
  |-- Stage 5: Monte Carlo Yield Distribution
  |-- Stage 6: DRC Rule Compilation
  |
Output: Yield prediction, risk maps, DRC violations, HTML report
```

---

## Stage 1: Feature Extraction

**Purpose:** Extract spatial features from the layout that drive downstream physics.

**Algorithm:**
1. Parse GDS/OASIS file using gdstk library, selecting the specified pad layer.
2. Rasterize pad geometries onto a regular tile grid (default 25 um tile size).
3. Compute pad density per tile (fraction of tile area covered by Cu pads).
4. Compute density gradient magnitude (finite differences).
5. Detect motif patterns: isolated pads, TSV proximity zones, extreme gradient regions.

**Output:** `FeatureSet` dataclass containing density map, gradient map, motif masks, pad count, bounding box.

---

## Stage 2: CMP Recess Prediction

**Purpose:** Predict post-CMP surface topography from pad density patterns.

**Physics basis:** Chemical-mechanical polishing removes material at a rate governed by the Preston equation: removal rate is proportional to local pressure times relative velocity. For patterned wafers, local pressure depends on the effective pad density within the planarization length scale. This creates the well-known "bathtub curve" -- high recess (dishing) at very low density, minimum recess at moderate density (~50%), and increasing recess (erosion) at very high density.

**Algorithm:**
1. Convolve raw pad density with a Gaussian kernel whose sigma is derived from the Effective Planarization Length (EPL, default 100 um). The EPL represents the spatial averaging scale of the CMP process.
2. Map effective density to recess (nm) via PCHIP interpolation (Piecewise Cubic Hermite Interpolating Polynomial). PCHIP preserves monotonicity on each segment and avoids Runge-type overshoot.
3. Compute density-dependent recess sigma: higher variation at low density where isolated features are sensitive to local process variation.
4. Derive gap proxy: recess plus surface roughness.
5. Compute CMP margin index: gap / threshold, clipped to [0, 1].

**Calibration:** Default 5-point bathtub curve from Stine et al. 1998 (aluminum CMP). Must be replaced with fab-specific Cu CMP measurements for production use.

**Reference:** Stine et al., IEEE Trans. Semicond. Manuf., 1998.

---

## Stage 3: Bond Void Risk (Contact Mechanics)

**Purpose:** Determine whether the post-CMP surface topography allows intimate contact during wafer bonding, or whether gaps remain that become trapped voids.

**Physics basis:** When two wafers are brought into contact, the combination of surface adhesion (van der Waals forces at the bond interface) and wafer elasticity determines the equilibrium contact state. Stiff, thick wafers with weak adhesion will bridge over topographic features, leaving voids. Flexible, thin wafers with strong adhesion will conform to the topography.

**Algorithm (Spectral FFT solver):**
1. Optionally refine the tile grid to sub-um elements via bilinear interpolation.
2. Build the elastic response kernel in Fourier space: plate bending stiffness (D * K^4) and surface compliance (E*/(2*dx)) in serial compliance. This kernel is diagonal in Fourier space.
3. Model adhesion using the Dugdale cohesive zone: constant traction sigma_max = gamma/dc for gap < dc, zero for gap > dc. A smoothed Huber-like softplus approximation (alpha=20) provides numerical stability.
4. Minimize the total energy (elastic + adhesion) using L-BFGS-B with box constraints.
5. Compute per-tile void risk from the equilibrium gap field.

**Complexity:** O(N log N) per L-BFGS-B iteration via FFT. Typical convergence in 50-200 iterations.

**Reference:** Turner & Spearing, J. Appl. Phys., 2002. Maugis, J. Colloid Interface Sci., 1992.

---

## Stage 4: Anneal Delamination Risk (Thermal Stress)

**Purpose:** Compute thermal stress from CTE mismatch during post-bond annealing and assess delamination risk.

**Physics basis:** Copper (CTE = 17 ppm/K) and silicon dioxide (CTE = 0.5 ppm/K) expand at very different rates during the 200-400C anneal used to strengthen hybrid bonds. This CTE mismatch creates stress at material interfaces. If the elastic strain energy release rate exceeds the interface fracture toughness, delamination occurs.

**Algorithm:**
1. Assign composite material properties per tile using Voigt rule-of-mixtures: density=1 maps to pure Cu, density=0 maps to pure SiO2.
2. Compute plane-stress stiffness from composite E and nu.
3. Compute thermal strain: eps_th = CTE * delta_T (where delta_T = anneal_temp - room_temp).
4. Assemble and solve the 2D plane-stress equilibrium equations using sparse direct solver (CPU) or conjugate gradient (GPU).
5. Compute von Mises equivalent stress from the stress tensor.
6. Assess delamination risk using fracture mechanics interface flaw model.

**Limitation:** Linear elastic only. No plasticity, creep, or stress relaxation.

**Reference:** Suhir, J. Appl. Mech., 1986.

---

## Stage 5: Monte Carlo Yield Distribution

**Purpose:** Propagate process parameter uncertainty through the physics chain to produce yield distributions.

**Physics basis:** Die yield is a function of the spatial distribution of failure probabilities across the die. The Murphy/Stapper negative-binomial model accounts for spatial clustering of defects, which is a well-documented phenomenon in semiconductor manufacturing.

**Algorithm:**
1. For each Monte Carlo trial (default 200 trials):
   a. Sample process parameters from their distributions: overlay sigma, particle density, surface roughness.
   b. Perturb risk maps with sampled parameters.
   c. Compute per-tile failure probability: P_fail = 1 - (1-P_open)(1-P_void)(1-P_delam).
   d. Apply edge exclusion (default 2 tiles per edge).
   e. Partition die interior into spatial clusters based on correlation length.
   f. Per-cluster yield: Y_k = 1/(1+D_k) (Murphy model).
   g. Die yield = product of cluster yields times Poisson particle yield.
2. Compute summary statistics: P10, P50, P90, mean.
3. Run sensitivity analysis: quartile-based elasticity for each parameter.

**Critical parameter:** `correlation_length_um` (default 500 um) dominates the final yield prediction. Must be calibrated to fab-specific data.

**References:** Murphy, Proc. IEEE, 1964. Stapper, IBM J. Res. Dev., 1983.

---

## Stage 6: DRC Rule Compilation

**Purpose:** Translate physics solver outputs into layout-level design rule violations.

**Algorithm:**
1. Apply threshold rules to each solver output:
   - Minimum density (default 0.15)
   - Maximum density (default 0.85)
   - Maximum density gradient (default 0.10)
   - Void risk threshold (default 0.7)
   - Delamination risk threshold (default 0.7)
   - CMP margin critical threshold (default 0.8)
2. Generate binary violation masks (per tile).
3. Generate actionable suggestions with severity ranking.
4. Export KLayout marker database (.lyrdb) for visualization.

---

## Inverse Design Optimizer (Optional)

**Purpose:** Modify fill density patterns to jointly minimize bond void risk, delamination risk, CMP non-uniformity, and thermal stress.

**Two strategies:**

**Additive (Hill Climbing):**
- Identify highest combined-risk tile.
- Place Gaussian fill blob at hotspot.
- Accept if multi-objective fitness improves.
- Multi-start restarts to avoid local minima.

**Subtractive (Stress Minimizing):**
- Start from checker fill base.
- Run thermal solver to find stress hotspots.
- Carve out fill at peak stress location.
- Accept only if yield maintained and stress improved.

**Multi-objective fitness:** Weighted combination of void risk, delamination risk, CMP sigma, and thermal peak stress.

---

## Material Properties

| Property | Copper (Cu) | Silicon Dioxide (SiO2) | Silicon (Si) |
|---|---|---|---|
| Young's modulus (GPa) | 130 | 70 | 170 |
| Poisson ratio | 0.34 | 0.17 | 0.28 |
| CTE (ppm/K) | 17.0 | 0.5 | 2.6 |
| Yield strength (MPa) | ~300 (at temp) | N/A (brittle) | N/A |

---

*This document describes the solver architecture at a conceptual level. Source code is not included in this public repository.*
