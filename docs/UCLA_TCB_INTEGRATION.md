# Integration Brief: UCLA Two-Stage TCB + Genesis Yield Prediction

> UCLA has the process physics (how to bond fast).
> Genesis has the prediction physics (whether it will work before you try).
> Together: a complete heterogeneous integration platform.

---

## The Opportunity

UCLA's high-throughput thermal compression bonding process (Sahoo, Ren, Iyer --
ECTC 2023, DOI: 10.1109/ECTC51909.2023.00067) achieves approximately 900 units
per hour at <=10um pitch using a two-stage approach: rapid tacking followed by
batch anneal. This is among the highest demonstrated throughputs for fine-pitch
Cu-Cu bonding in an academic setting.

Genesis Bondability provides the **pre-fabrication yield prediction layer** that
wraps around this process. The bonding physics are UCLA's contribution. The
design-stage prediction of whether a given layout and process window will produce
acceptable yield -- before committing wafers -- is ours.

No commercial tool currently provides this capability. EDA tools model CMP.
Metrology tools measure overlay after bonding. But nobody predicts yield from
layout parameters before the bonding step. That is what Genesis does.

## What Genesis Adds to UCLA TCB

### 1. Pre-Bonding Yield Prediction

Given a layout (pad density map, die dimensions, material stack) and process
parameters (anneal temperature, overlay sigma, surface roughness), Genesis predicts
yield P10/P50/P90 distributions *before* the 900 UPH line runs. The full physics
chain executes in under 1 second per layout.

### 2. Process Window Mapping

Genesis maps the safe operating zone across the process parameters UCLA controls:
tacking temperature, anneal time and temperature, bonding pressure, overlay tolerance.
A 10,000-point Latin Hypercube design of experiments runs in under 3 hours on a
single CPU, producing a complete process window with 85.9% mean yield across the
explored space.

### 3. KLA Metrology Bridge

Post-bond overlay measurements from KLA Archer (or equivalent tool) feed directly
into the Genesis pipeline. Overlay dx/dy maps are converted to spatial autocorrelation
estimates, which update the yield prediction via Bayesian calibration. This creates
a closed-loop system: bond, measure, predict, adjust.

### 4. Bayesian Calibration

Genesis converges from initial physics-based predictions to fab-accurate yield
estimates in approximately 10 wafers (vs. 100+ with manual process tuning). The
Bayesian update uses KLA metrology as the observation model and the physics pipeline
as the prior.

## Technical Compatibility

| UCLA TCB Parameter | Genesis Coverage | Notes |
|--------------------|-----------------|-------|
| Cu-Cu bonding at <=10um pitch | Supported | Spectral FFT contact solver handles fine pitch |
| 295-302C anneal window | Supported | Thermal stress module covers this range |
| Die-to-wafer assembly | Supported | Layout-aware yield prediction per die position |
| Two-stage process (tack + anneal) | Supported | Sequential thermal profile modeling |
| ~900 UPH throughput | Compatible | <1s prediction per layout; does not bottleneck |
| MIL-STD-883 shear strength | Partial | Elastic model only -- no plasticity regime |
| Al-to-Cu heterogeneous bonding | Partial | Calibration needed for non-Cu-Cu metallurgy |

**Supported** = validated in current pipeline.
**Partial** = architecture supports it, but calibration or model extension needed.

## The Value Proposition

UCLA's Si-IF (Silicon Interconnect Fabric) program demonstrates that high-throughput
heterogeneous integration at fine pitch is physically achievable. Genesis demonstrates
that yield prediction for these processes can be computed from first principles in
under 1 second.

The combination creates something neither party has alone: **a design-for-manufacturing
platform for heterogeneous integration** where process engineers can evaluate layout
candidates against bonding yield *before* committing to fabrication runs.

For UCLA's industry partners and licensees, this means faster process qualification,
fewer wasted wafers during ramp, and quantitative confidence in yield projections
that currently rely on engineering intuition.

## Proposed Collaboration

1. **License Genesis for the Si-IF program.** UCLA researchers use the yield prediction
   pipeline to evaluate layout candidates before bonding runs, reducing the experimental
   design space.

2. **Co-publish at ECTC 2026.** Joint paper demonstrating physics-based yield prediction
   validated against UCLA TCB experimental data. First published demonstration of
   pre-fab yield prediction for fine-pitch hybrid bonding.

3. **Joint IP for process-specific calibration.** UCLA provides experimental yield data
   for the two-stage TCB process. Genesis provides the calibration framework. Joint
   ownership of the resulting process-specific prediction models.

4. **Bayesian calibration validation.** Use UCLA's wafer-level data to validate the
   10-wafer convergence claim and establish calibration protocols for academic and
   industry adoption.

## Related UCLA Publications

- **Sahoo, S.K., Ren, H., Iyer, S.S.** "High-Throughput Thermal Compression Bonding
  for Heterogeneous Integration," *73rd IEEE ECTC*, 2023.
  DOI: [10.1109/ECTC51909.2023.00067](https://doi.org/10.1109/ECTC51909.2023.00067)

- **Ren, H.** "Thermal Compression Bonding for Fine-Pitch Cu-Cu Interconnects,"
  *Ph.D. Dissertation, UCLA*, 2021.

- **Sahoo, S.K., Ren, H., Iyer, S.S.** "Al-Cu Heterogeneous Bonding for Heterogeneous
  Integration," *74th IEEE ECTC*, 2024.

- **Sahoo, S.K.** "High-Throughput Bonding and Assembly for Silicon Interconnect Fabric,"
  *Ph.D. Dissertation, UCLA*, 2025.

- **Iyer, S.S.** "Silicon Interconnect Fabric -- A Platform for Heterogeneous Integration,"
  *IEEE IEDM*, 2019.

## Contact

**Nicholas Harris** -- [nick@nmk.ai](mailto:nick@nmk.ai)

We are actively seeking academic and industry partners for calibration validation.
If your group has experimental bonding yield data and wants physics-based prediction
capability, please reach out.

---

*This document is part of the Genesis PROV 9: Bondability public data room.
See [HONEST_DISCLOSURES.md](../HONEST_DISCLOSURES.md) for limitations and caveats.*
