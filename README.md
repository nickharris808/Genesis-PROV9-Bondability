# Bondability: Pre-Fab Yield Prediction for Hybrid Bonding

> Predict hybrid bonding yield from design parameters -- before you commit wafers.
> The only GDS-to-yield pipeline for Cu-Cu direct bonding at <=10um pitch.

---

## The Problem

Hybrid bonding (Cu-Cu direct bonding) is the critical interconnect technology for advanced
3D stacking -- HBM, chiplets, CoWoS, Foveros. Every major foundry is racing to production
at <=10um pitch. But yield prediction is still trial-and-error: you design a layout,
fabricate wafers, bond them, and *then* discover whether your density patterns, CMP recess,
and process window produce acceptable yield. Each iteration costs $50K+ in wafer starts
and takes weeks to months.

There is no commercial tool that predicts hybrid bonding yield from layout parameters
before fabrication. EDA vendors model CMP planarity. Metrology vendors measure overlay
after bonding. But nobody closes the loop from GDS layout to yield prediction in a single
pipeline. That gap costs the industry millions per tape-out in wasted wafers and delayed
ramps.

## What This Does

Bondability is a 7-stage physics pipeline that predicts hybrid bonding yield from design
inputs in under 1 second per layout:

```
1. Feature Extraction     -- GDS/OASIS layout -> pad density map, motifs, gradients
2. CMP Recess Prediction  -- Preston equation + EPL kernel -> surface topography (nm)
3. Spectral Contact        -- FFT contact solver + Dugdale adhesion -> bond/void map
4. Void Formation          -- Logistic void risk from gap vs. snap-in threshold
5. Anneal Thermal Stress   -- CTE-mismatch FEA -> interfacial stress + delamination risk
6. Murphy Yield Model      -- Spatial defect correlation -> P10/P50/P90 yield distribution
7. DRC Rule Compilation    -- Physics-derived design rules + KLayout marker output
```

100 out of 100 test layouts run through the full pipeline with zero errors.
Every solver is benchmarked against published data (Stine 1998, Turner 2002, Suhir 1986, Murphy 1964).

## Why It Matters for Advanced Packaging

UCLA's two-stage thermal compression bonding process (Sahoo, Ren, Iyer -- ECTC 2023)
achieves ~900 units per hour at <=10um pitch. Intel Foveros is ramping direct bonding
for next-generation chiplet architectures. TSMC CoWoS-L is scaling to larger interposer
sizes with hybrid bonding. Samsung is pushing 3D DRAM stacking with Cu-Cu bonds.

All of these processes need yield prediction that keeps pace with throughput. If you are
bonding dielets at 900 UPH with thermal compression, you need to know *before* the line
runs whether a given layout and process window will produce acceptable yield. Our pipeline
runs the full physics chain in under 1 second -- faster than the tacking step.

See [UCLA TCB Integration Brief](docs/UCLA_TCB_INTEGRATION.md) for how Bondability
wraps around high-throughput bonding processes.

## Key Results (Verified)

| Metric | Value | Evidence |
|--------|-------|----------|
| Layouts validated | 100/100 | Full pipeline, zero errors |
| Mean yield (P50) | 0.60 | 4 patterns x 3 densities x 3 oxide thicknesses |
| KLA calibration convergence | 10 wafers -> CI<20um | Bayesian DOE (10K campaigns) |
| Process window (10K LHS) | 85.9% mean yield | Full physics, 0 errors |
| FNO inference speed | 13ms/die (CPU) | Neural surrogate fast path |
| CMP benchmark (Stine 1998) | PASS | Hold-out validation at 6 densities |
| Contact benchmark (Turner 2002) | PASS | 4 qualitative physics checks |
| Thermal benchmark (Suhir 1986) | PASS | Analytical bimetallic comparison |

All results are from computational benchmarks. See [evidence/key_results.json](evidence/key_results.json)
for raw data and [verification/verify_claims.py](verification/verify_claims.py) to reproduce.

## Integration Points

- **KLA Archer:** Overlay CSV -> yield prediction in <100ms. Every Archer installation
  becomes a yield prediction node. See [KLA Integration Brief](docs/KLA_INTEGRATION_BRIEF.md).
- **TSMC / Intel / Samsung:** GDS-in, yield-out. API or on-premise deployment.
  See [Deployment Checklist](docs/DEPLOYMENT_CHECKLIST.md).
- **UCLA TCB:** Pre-bonding yield prediction for two-stage die-to-wafer assembly at 900 UPH.
  See [UCLA TCB Integration](docs/UCLA_TCB_INTEGRATION.md).
- **Corning / AGC:** Glass substrate bonding yield -- extends to glass interposer programs.

## Patent Portfolio

- **13 claims filed** (provisional, January 2026)
- **7 additional claims** in filing pipeline
- **909 total claims** across the Genesis platform (9 provisionals covering advanced packaging,
  glass PDK, smart matter, chiplet assembly, yield screening, and more)
- See [CLAIMS_SUMMARY.md](CLAIMS_SUMMARY.md) for claim details and valuation estimates.

## Honest Limitations

We practice radical transparency. See [HONEST_DISCLOSURES.md](HONEST_DISCLOSURES.md) for
the full list of 15 documented limitations.

Key caveats an investor or partner should know:

- **Computational only** -- no fabricated wafers, no fab process data collected
- **CMP model needs calibration** -- default is from published literature, not fab-specific
- **correlation_length dominates yield** -- Sobol S1 = 0.582; this single parameter swings
  yield by 20+ percentage points and must be calibrated per-fab
- **FNO surrogate R-squared = 0.50** -- screening quality only, not production prediction
  (the previously reported 0.8725 has been retracted; see [SCIENCE_NOTES.md](SCIENCE_NOTES.md))
- **Linear elastic only** -- no plasticity modeling at anneal temperatures

These are not bugs. They are the honest state of a research platform that has the right
physics architecture but needs fab partner data to calibrate. The architecture is the
hard part. Calibration is the partnership opportunity.

## Verify Our Claims

Every claim in this repository can be independently verified:

```bash
pip install -r verification/requirements.txt
python verification/verify_claims.py
```

Requires Python 3.10+. No proprietary packages. No GPU. No fab data.
See [Reproduction Guide](docs/REPRODUCTION_GUIDE.md) for details.

## What's Validated and What's Not

See [Validation Summary](docs/VALIDATION_SUMMARY.md) for the complete scope:
what materials, geometries, and process conditions are covered -- and what falls
outside the validated envelope.

## Licensing

| Tier | Access | Price |
|------|--------|-------|
| **Free** | This repository, verification script, documentation | $0 |
| **SaaS Starter** | API access, 1,000 layouts/month | $500/month |
| **SaaS Professional** | API access, unlimited layouts, priority support | $2,500/month |
| **Enterprise** | On-premise deployment, custom calibration, source access | Custom |

Patent license is separate from software license. 909 claims across 9 provisionals.
Contact for licensing terms.

## Contact

**Nicholas Harris** -- [nick@nmk.ai](mailto:nick@nmk.ai)

[Request Demo](https://nmk.ai/contact) | [View Genesis Platform](https://nmk.ai)

---

## Repository Structure

```
README.md                           <-- You are here (business overview)
CLAIMS_SUMMARY.md                   Patent claims with valuation + filing status
HONEST_DISCLOSURES.md               15 documented limitations (radical transparency)
SCIENCE_NOTES.md                    Audit findings and scientific notes
LICENSE                             MIT

docs/
  TECHNICAL_INVENTORY.md            Detailed technical inventory (for DD readers)
  SOLVER_OVERVIEW.md                Conceptual solver architecture
  REPRODUCTION_GUIDE.md             How to run verification
  UCLA_TCB_INTEGRATION.md           UCLA partnership integration brief
  KLA_INTEGRATION_BRIEF.md          KLA Archer yield bridge
  DEPLOYMENT_CHECKLIST.md           Sanitized deployment guide
  VALIDATION_SUMMARY.md             What's validated, what's not

evidence/
  key_results.json                  Benchmark results (machine-readable)

verification/
  verify_claims.py                  Standalone verification script
  verification_results.json         Latest verification output
  requirements.txt                  Python dependencies (no proprietary packages)
  reference_data/
    canonical_values.json           Reference values for verification
```

For the complete technical inventory (directory trees, file listings, CLI reference,
API reference, solver details), see [docs/TECHNICAL_INVENTORY.md](docs/TECHNICAL_INVENTORY.md).
That document is 1,500+ lines of detailed technical documentation for due diligence readers
who want depth beyond this overview.

---

*Zero source code. Zero training data. Zero calibration secrets. All honest disclosures preserved.*
