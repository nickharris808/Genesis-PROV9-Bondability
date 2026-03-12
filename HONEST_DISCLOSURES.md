# Genesis PROV 9: Bondability -- Honest Disclosures

This document describes the limitations, caveats, and honest status of the Bondability research platform. It is intended to prevent misunderstanding of what has and has not been demonstrated.

---

## 1. Research Status -- Not a Provisional Patent

Bondability is research-stage software. The 13 claims described in CLAIMS_SUMMARY.md are research claims articulating the technical contributions. **No provisional patent application has been filed.** The claims have not been reviewed by patent counsel and do not constitute legal patent claims.

---

## 2. Computational Only -- No Fabricated Wafers

No bonded wafers have been fabricated as part of this work. No fab process data has been collected. All results are computational predictions from physics-based models. The platform has not been tested against real manufacturing outcomes.

This is a fundamental limitation. Computational models, no matter how physically grounded, must ultimately be validated against experimental data from a specific fab process. Until that calibration step occurs, all yield predictions are directional (correct physics, uncalibrated magnitude).

---

## 3. Validated Against Published Benchmarks, Not Proprietary Fab Data

The four validation benchmarks use published academic data:

| Benchmark | Reference | Data Type |
|---|---|---|
| CMP recess prediction | Stine et al. 1998, IEEE TSM | Published aluminum CMP data |
| Contact mechanics | Turner & Spearing 2002, J. Appl. Phys. | Published wafer bonding theory |
| Thermal stress | Suhir 1986, J. Appl. Mech. | Analytical bimetallic solution |
| Yield model | Murphy 1964, Proc. IEEE | Published yield theory |

None of these benchmarks use proprietary data from TSMC, Samsung, Intel, or any other foundry. The benchmarks verify that the solvers implement the published physics correctly. They do not verify that the models predict the yield of a specific fab process.

---

## 4. CMP Calibration Is from Aluminum, Not Copper

The default CMP calibration curve uses data from Stine et al. 1998, which characterized **aluminum** chemical-mechanical polishing. The qualitative behavior (bathtub-shaped recess vs. density curve, with minimum recess near 50% density) is similar for copper damascene CMP, but the absolute recess values, EPL lengths, and density-dependence coefficients differ.

**This is the single most critical calibration item for any production deployment.** The platform includes a calibration module for loading fab-specific Cu CMP measurements.

---

## 5. Spectral FFT Is a Standard Technique

The spectral (Fourier-domain) approach to contact mechanics is well-established in the tribology and surface science literature. Key references include:

- Persson, B.N.J., "Contact Mechanics for Randomly Rough Surfaces," Surf. Sci. Rep., 2006
- Greenwood, J.A. and Williamson, J.B.P., "Contact of Nominally Flat Surfaces," Proc. R. Soc. London A, 1966
- Stanley, H.M. and Kato, T., "An FFT-Based Method for Rough Surface Contact," J. Tribol., 1997

Our contribution is not the invention of spectral contact mechanics. It is the application of spectral FFT contact solving within an end-to-end hybrid bonding yield prediction pipeline, combined with Dugdale cohesive zone adhesion modeling specific to the Cu-Cu bonding problem.

---

## 6. Linear Elastic Thermal Solver -- No Plasticity

The thermal stress FEA solver computes **linear elastic** stress only. It does not model:

- Copper plasticity (yield at ~300 MPa at anneal temperatures)
- Stress relaxation via creep
- Viscoelastic behavior of dielectric materials
- Interface sliding or friction

At 300C anneal with sharp Cu/SiO2 interfaces, the solver predicts peak von Mises stress that can exceed Cu yield strength. In reality, copper would yield plastically and stress would redistribute. The solver is therefore **conservative** (overestimates interfacial stress), which is the safe-side error for a screening tool.

Implementing proper incremental plasticity (load stepping, tangent stiffness updates, deviatoric radial return mapping) was a deliberate scope decision. It would add significant complexity for a modest improvement in stress accuracy at design-stage screening resolution.

---

## 7. Correlation Length Dominates Yield Predictions

The yield model's `correlation_length_um` parameter controls the number of independent failure clusters used in the Murphy/Stapper calculation:

```
n_clusters = (die_height * die_width) / (correlation_length^2)
```

This is the single most impactful parameter on final yield numbers. Varying it from 200 um to 2000 um can change predicted yield by 20+ percentage points. The default value (500 um) is a reasonable starting point based on literature, but it is **not calibrated to any specific fab's defect correlation data.**

Any quantitative yield prediction requires calibrating this parameter against actual fab yield measurements.

---

## 8. Contact Solver Under-Resolution at Tile Scale

The Dugdale cohesive zone width (dc = 5 nm) operates at nanometer scale, but the default tile grid resolution is 25 um. At tile resolution, the solver produces qualitatively correct bridging/conforming behavior but quantitative gap values carry 2-5x error. The solver issues a warning about this.

For quantitative accuracy, mesh refinement (`refine_factor >= 4`) subdivides each tile into sub-elements, at the cost of 16x slower computation.

---

## 9. No Wafer-Level Warpage Computation

The platform uses a simple `warpage_um` input parameter but does not compute wafer warpage from first principles. A proper warpage model would require:

- Wafer-scale (300mm) FEA
- Full film stack stress integration
- Thermal gradient effects
- Chuck clamping mechanics

This is a significant modeling gap for die-to-wafer (D2W) bonding, where wafer bow directly affects contact pressure distribution.

---

## 10. Heuristic Motif Detection

The feature extraction module detects "bad motifs" (isolated pads, TSV proximity regions, extreme gradient zones) using threshold-based heuristics on the density map. It does not perform true pattern recognition and cannot identify specific layout structures such as guard rings, seal rings, or specific dummy fill geometries.

---

## 11. ML Surrogate Is Experimental

The ML surrogate module exists in the codebase but is experimental. It is NOT used in the production physics pipeline. The training dataset is small (200 samples) and the surrogate has not been validated for accuracy against the physics solvers across the full parameter space.

---

## 12. No Multi-Layer CMP Interaction

The CMP model is single-layer (2D effective density). It does not model:

- Multi-layer CMP interactions (dishing propagation through stacked layers)
- Through-silicon via (TSV) topography effects on CMP
- Inter-layer density coupling

For multi-layer 3D stacks (e.g., 12-high HBM), each bonding interface would need independent CMP modeling, and inter-layer effects would need to be accounted for separately.

---

## 13. Test Suite Scope

The 17 test files cover:

- All solver modules (CMP, contact, thermal, bonding, anneal, yield)
- Pipeline integration (end-to-end execution)
- CLI command execution
- REST API endpoint validation
- Adversarial edge cases (zero density, extreme temperatures, NaN inputs)
- Published-data benchmarks

All tests pass. This demonstrates code quality, internal consistency, and correct implementation of published physics models. It does **not** demonstrate manufacturing readiness, fab-calibrated accuracy, or production deployment capability.

---

## 14. Monte Carlo Samples Default (200) Is Below Convergence Minimum

The default `monte_carlo_samples=200` throughout the codebase (config.py, process YAML files, validation benchmarks) is below the generally accepted minimum of 1,000 samples for Monte Carlo convergence of yield distributions. At 200 samples, P10/P90 quantile estimates have high variance and are not statistically reliable. The yield_p50 (median) is more robust but still carries significant sampling noise.

**Users should increase `monte_carlo_samples` to at least 1,000 (preferably 5,000-10,000) for any quantitative yield prediction.** The default of 200 was chosen for demo speed, not statistical rigor.

---

## 15. Void Risk Model Behavior at Low Density

The logistic void risk model (`p_void = 1/(1 + exp(-k*(gap - snap_in)))`) produces p_void approaching 1.0 for all tiles when the surface gap significantly exceeds the snap-in threshold (~9 nm). This is physically correct -- if the surface gap is much larger than the adhesion pull-in distance, bonding cannot occur. However, this means:

- Layouts with very low pad density (e.g., <5%) will produce gap >> snap_in, resulting in p_void near 1.0 for every tile, and therefore yield near 0.0.
- This is not a bug but rather the correct prediction for unrealistic layouts.
- For layouts near the snap-in threshold, small changes in CMP recess or roughness can swing p_void dramatically, which is the root cause of the correlation_length sensitivity documented in Disclosure #7.

The logistic steepness parameter `k_void_logistic = 2.0` was chosen to represent the statistical ensemble of surface roughness asperities. It is not calibrated to experimental void rate data.

---

*This disclosure document is part of the non-confidential public repository for Genesis PROV 9: Bondability. It is intended to provide a candid, honest assessment of the current state of the research platform.*
