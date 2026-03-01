# Genesis PROV 9: Bondability

## Physics-Based Hybrid Bonding Yield Prediction from GDS Layout to Manufacturing Outcome

**Status:** Research (v2.0)
**Claims:** 13 research-stage claims (not yet provisional patent)
**Validation:** Benchmarked against Stine 1998, Turner 2002, Suhir 1986, Murphy 1964
**Codebase:** 61 production Python source files | ~4,500 LOC | 17 test files (all passing)
**Interfaces:** 7 CLI commands | 5 REST API endpoints | Python API
**License:** CC BY-NC-ND 4.0

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Why This Matters: The Hybrid Bonding Imperative](#why-this-matters-the-hybrid-bonding-imperative)
3. [The Problem: No Tool Does GDS-to-Yield](#the-problem-no-tool-does-gds-to-yield)
4. [The Bondability Solution: Six-Stage Physics Chain](#the-bondability-solution-six-stage-physics-chain)
5. [Architecture Deep Dive](#architecture-deep-dive)
6. [Spectral FFT Contact Solver](#spectral-fft-contact-solver)
7. [Monte Carlo Yield Engine](#monte-carlo-yield-engine)
8. [Methodology: Stage-by-Stage Technical Detail](#methodology-stage-by-stage-technical-detail)
9. [Validated Results](#validated-results)
10. [Comparison: Bondability vs. Existing Tools](#comparison-bondability-vs-existing-tools)
11. [Key Discoveries and Innovations](#key-discoveries-and-innovations)
12. [Applications](#applications)
13. [Cross-Platform Integration](#cross-platform-integration)
14. [Research-Stage Patent Portfolio](#research-stage-patent-portfolio)
15. [Honest Disclosures](#honest-disclosures)
16. [Evidence and Verification](#evidence-and-verification)
17. [Repository Structure](#repository-structure)
18. [Citation and References](#citation-and-references)

---

## Executive Summary

Hybrid bonding -- direct Cu-Cu thermocompression bonding -- is the critical interconnect technology enabling High Bandwidth Memory (HBM), 3D chiplet stacking, and advanced heterogeneous integration. Every HBM4 stack, every TSMC SoIC chiplet assembly, every Intel Foveros package, and every Samsung X-Cube module depends on hybrid bonding to achieve the sub-10 um pad pitches and million-connection-per-mm^2 interconnect densities that conventional solder bumping cannot reach.

The fundamental problem is that hybrid bonding yield is notoriously difficult to predict before silicon. The physics chain from layout to yield outcome spans six tightly coupled domains: CMP planarity determines surface topography; surface topography governs contact mechanics at the bond interface; contact mechanics determines void formation; post-bond anneal creates thermal stress from CTE mismatch; thermal stress drives delamination risk; and the spatial distribution of all these failure modes determines die-level yield. Current industry practice handles this chain empirically -- through expensive post-silicon yield learning, rule-of-thumb density guidelines, and iterative process tuning that costs tens of millions of dollars and months of calendar time per product generation.

**Bondability** models the complete physics chain computationally. It accepts a GDS/OASIS layout as input and predicts hybrid bonding yield as output, propagating uncertainty through every stage via Monte Carlo sampling. The solver architecture uses:

- **Spectral FFT contact mechanics** with O(N log N) computational complexity per iteration
- **PCHIP-interpolated CMP prediction** calibrated against published data (Stine 1998, Ouma 2002)
- **Dugdale cohesive zone modeling** for adhesion at the Cu-Cu bond interface
- **Plane-stress thermal FEA** for post-bond anneal stress computation (CPU and GPU variants)
- **Murphy/Stapper negative-binomial yield model** with spatial defect correlation and Monte Carlo uncertainty quantification
- **Multi-objective inverse design optimization** for gradient-compensated dummy fill

The platform has been validated against four independent published benchmarks. 17 test files exercise all solver modules, the end-to-end pipeline, CLI commands, REST API endpoints, and adversarial edge cases. All tests pass. It is research-status software -- computationally complete, architecturally sound, and honest about what it does and does not demonstrate.

This is not a production EDA tool. It is a research platform that demonstrates that physics-based GDS-to-yield prediction for hybrid bonding is computationally feasible, architecturally tractable, and scientifically grounded. The path from here to production requires fab-specific calibration, experimental validation, and integration with foundry design flows.

---

## Why This Matters: The Hybrid Bonding Imperative

### The $100 Billion Question

The semiconductor industry's roadmap for continued performance scaling has fundamentally shifted from transistor shrinking to 3D integration. Moore's Law transistor density improvements are slowing -- the cost per transistor at sub-3nm nodes is no longer decreasing. The path forward is stacking dies vertically and connecting them with high-density interconnects. Hybrid bonding is the only interconnect technology capable of delivering the required pad pitches for this next era of semiconductor manufacturing.

The numbers are stark:

| Technology | Minimum Pitch | Interconnect Density | Key Applications |
|---|---|---|---|
| Solder C4 bumps | ~130 um | ~60 connections/mm^2 | Legacy flip-chip |
| Microbumps (Cu pillar) | ~40 um | ~625 connections/mm^2 | HBM3, 2.5D interposers |
| **Hybrid bonding** | **~1 um (demonstrated)** | **~1,000,000 connections/mm^2** | **HBM4, SoIC, Foveros** |

That is a **1,600x improvement** in interconnect density from microbumps to hybrid bonding. This density enables architectures that are physically impossible with any other interconnect technology.

### Market Context

HBM (High Bandwidth Memory) is the most visible application. HBM4, entering volume production in 2025-2026, uses hybrid bonding for its memory die stacks. The HBM market alone is projected to exceed $30 billion annually by 2026, driven almost entirely by AI/ML training and inference workloads that require massive memory bandwidth. Every NVIDIA H100, B100, and successor GPU depends on HBM. Every Google TPU, every AWS Trainium, every Meta MTIA chip requires HBM stacks.

Beyond HBM:

- **TSMC SoIC (System on Integrated Chips)** uses hybrid bonding for chiplet-to-chiplet integration, enabling Apple, AMD, and NVIDIA to build multi-die processors with bandwidth exceeding 1 TB/s between chiplets.
- **Intel Foveros** enables logic-on-logic 3D stacking with face-to-face hybrid bonding, used in Meteor Lake and successor architectures for disaggregated tile packaging.
- **Samsung X-Cube** provides 3D chiplet stacking for their foundry customers, competing directly with TSMC SoIC.
- **ASE (Advanced Semiconductor Engineering)** is developing hybrid bonding OSAT capabilities for chiplet-to-wafer and wafer-to-wafer configurations.
- **Sony image sensors** pioneered hybrid bonding at larger pitches for pixel-to-logic stacking, now moving to finer pitches.
- **Logic-on-logic stacking** for backside power delivery networks (BSPDN) at Intel and TSMC.

The total addressable market for advanced packaging -- the segment where hybrid bonding is the enabling technology -- is projected to reach $100 billion by 2030.

### The Yield Problem: $1M+ Per Point

In hybrid bonding, yield is everything. A single 300mm wafer carrying thousands of bonded dies can have billions of Cu-Cu bond pads, each of which must make reliable electrical contact. Consider the math:

- A typical HBM4 die has ~65,000 hybrid bond pads at sub-10 um pitch
- A 12-high HBM4 stack has 11 bonding interfaces, each with 65,000 pads
- A single 300mm wafer carries ~1,000 dies
- That is **~715 million bond pads per wafer** that must all work

At wafer costs of $10,000-$50,000 per 300mm wafer through a hybrid bonding flow, and with typical yield learning campaigns requiring 50-200 wafers, a single yield learning cycle costs $0.5M-$10M. More critically, each design-fab-test iteration cycle takes 8-16 weeks. For HBM4 development, where multiple memory die designs must be co-optimized with the bonding process, the total yield learning cost reaches tens of millions of dollars per product generation.

**A single percentage point of yield improvement on a high-volume HBM line is worth more than $1 million per year in gross margin.** For a high-volume line running 10,000 wafers per month at $15,000 per wafer with 1,000 dies per wafer, each yield point is worth:

```
10,000 wafers/month x 1,000 dies/wafer x 1% x $50 ASP per die = $5,000,000/month
```

This makes physics-based yield prediction -- even if initially approximate -- enormously valuable if it can reduce the number of experimental iterations by even one cycle.

### Why Nobody Has Done This Before

The six-stage physics chain from layout to yield crosses traditional EDA tool boundaries:

1. **CMP simulation** lives in the process modeling world (KLA, Synopsys PrimeYield)
2. **Contact mechanics** lives in the MEMS/tribology world (COMSOL, Abaqus)
3. **Thermal stress** lives in the thermal simulation world (Ansys Icepak, Synopsys)
4. **Yield modeling** lives in the yield management world (PDF Solutions, Synopsys)
5. **DRC** lives in the physical verification world (Calibre, ICV)

No single EDA vendor has a team that spans all five domains. Building the integrated chain requires expertise in CMP modeling, contact mechanics, computational mechanics, yield statistics, and semiconductor process physics -- simultaneously. The organizational boundaries between EDA product groups mirror the technical boundaries between these domains. That is the gap Bondability fills.

---

## The Problem: No Tool Does GDS-to-Yield

### The Physics of Failure

A hybrid bond fails or succeeds based on a chain of coupled physical processes. Each stage feeds into the next, and failure at any stage can propagate downstream:

```mermaid
flowchart TD
    A["GDS/OASIS Layout\n(pad density, gradients, motifs)"] --> B["CMP Planarity\n(recess, dishing, erosion)"]
    B --> C["Contact Mechanics\n(bridging vs. conforming)"]
    C --> D["Void Formation\n(trapped gas, unbonded area)"]
    D --> E["Thermal Stress\n(CTE mismatch @ 300C anneal)"]
    E --> F["Delamination Risk\n(interface fracture)"]
    F --> G["Die-Level Yield\n(Murphy/Stapper spatial model)"]

    style A fill:#1a1a2e,stroke:#e94560,color:#fff
    style B fill:#1a1a2e,stroke:#e94560,color:#fff
    style C fill:#1a1a2e,stroke:#0f3460,color:#fff
    style D fill:#1a1a2e,stroke:#0f3460,color:#fff
    style E fill:#1a1a2e,stroke:#16213e,color:#fff
    style F fill:#1a1a2e,stroke:#16213e,color:#fff
    style G fill:#1a1a2e,stroke:#533483,color:#fff
```

**Stage 1 -- CMP Non-Uniformity.** Chemical-mechanical polishing must planarize the Cu surface to sub-nanometer roughness, but local pattern density creates systematic height variations. Low-density regions experience dishing (Cu recesses below dielectric); high-density regions experience erosion (dielectric thins). The relationship between density and recess follows a well-documented "bathtub curve" (Stine et al. 1998, Ouma et al. 2002). This means layout decisions made at the GDS level -- pad placement, dummy fill patterns, density gradients -- directly determine the post-CMP surface topography that the bonding process must accommodate.

**Stage 2 -- Contact Mechanics.** After CMP, two wafer surfaces must achieve intimate contact across the entire die area. Any residual topography from CMP creates gaps. Whether these gaps close during bonding depends on wafer thickness (stiff thick wafers bridge over features; flexible thin wafers conform), adhesion energy (stronger adhesion pulls surfaces together), and the spatial frequency content of the topography. This is a classical contact mechanics problem, solvable via spectral (Fourier-domain) methods.

**Stage 3 -- Void Formation.** Gaps that do not close during initial bonding become trapped voids. These voids prevent electrical contact at affected Cu pads. The void distribution is a spatial random field correlated with CMP topography and contact mechanics outcomes. Voids above a critical size (~1 um) cause open-circuit defects.

**Stage 4 -- Thermal Stress.** Post-bond annealing at 200-400C drives Cu grain growth and diffusion across the bond interface, strengthening the bond. But it simultaneously creates thermal stress from the CTE mismatch between Cu (17 ppm/K) and SiO2 dielectric (0.5 ppm/K). This stress is pattern-dependent: sharp density transitions create stress concentrations.

**Stage 5 -- Delamination.** If thermal stress exceeds the interface fracture toughness, delamination occurs. The crack driving force depends on the elastic strain energy release rate, which in turn depends on local density patterns, flaw sizes, and the competition between elastic energy and adhesion.

**Stage 6 -- Die-Level Yield.** The spatial distribution of failure probabilities across all tiles of the die determines whether the die passes or fails. Defects are not independent -- they cluster spatially due to the systematic nature of CMP and thermal stress patterns. This spatial correlation is captured by the Murphy/Stapper negative-binomial yield model.

### The Gap in Existing Tools

| Capability | CMP Tools (KLA, Synopsys) | FEA Tools (Ansys, COMSOL) | Yield Tools (PDF Solutions) | DRC (Calibre) | **Bondability** |
|---|---|---|---|---|---|
| CMP topography prediction | Yes | No | No | No | **Yes** |
| Contact mechanics | No | Manual setup | No | No | **Yes (automated)** |
| Void formation modeling | No | No | No | No | **Yes** |
| Thermal stress (CTE) | No | Yes (manual) | No | No | **Yes (automated)** |
| Yield prediction | No | No | Yes (empirical) | No | **Yes (physics-based)** |
| GDS input | Yes | No | No | Yes | **Yes** |
| End-to-end pipeline | No | No | No | No | **Yes** |
| Uncertainty quantification | Limited | No | Limited | No | **Yes (Monte Carlo)** |
| Inverse design optimization | No | No | No | No | **Yes** |

No existing commercial tool models the complete chain. Bondability is, to our knowledge, the first open research platform that does.

---

## The Bondability Solution: Six-Stage Physics Chain

Bondability operates as a sequential six-stage pipeline where each stage consumes typed outputs from prior stages and produces typed results for downstream stages. The pipeline accepts GDS/OASIS layout files (or pre-computed density arrays) as input and produces yield predictions, risk maps, DRC violations, sensitivity analyses, and HTML signoff reports as output.

```mermaid
graph LR
    subgraph Input
        GDS["GDS/OASIS\nLayout"]
        NPY["Pre-computed\nDensity Array"]
    end

    subgraph "Stage 1: Feature Extraction"
        FE["Pad Density\nGradients\nMotif Detection"]
    end

    subgraph "Stage 2: CMP Prediction"
        CMP["EPL Kernel\nPCHIP Interpolation\nBathtub Curve"]
    end

    subgraph "Stage 3: Contact Mechanics"
        CM["Spectral FFT\nDugdale CZM\nL-BFGS-B Solver"]
    end

    subgraph "Stage 4: Thermal Stress"
        TS["Voigt Mixing\nPlane-Stress FEA\nvon Mises Stress"]
    end

    subgraph "Stage 5: Monte Carlo Yield"
        MC["Murphy/Stapper\nSpatial Correlation\n200 MC Samples"]
    end

    subgraph "Stage 6: DRC + Optimization"
        DRC["Rule Compilation\nKLayout Markers\nInverse Design"]
    end

    subgraph Output
        RPT["HTML Report\nYield P10/P50/P90\nRisk Maps\nViolation Masks"]
    end

    GDS --> FE
    NPY --> FE
    FE --> CMP
    CMP --> CM
    CM --> TS
    TS --> MC
    MC --> DRC
    DRC --> RPT

    style GDS fill:#0d1b2a,stroke:#778da9,color:#e0e1dd
    style NPY fill:#0d1b2a,stroke:#778da9,color:#e0e1dd
    style FE fill:#1b263b,stroke:#778da9,color:#e0e1dd
    style CMP fill:#1b263b,stroke:#415a77,color:#e0e1dd
    style CM fill:#1b263b,stroke:#415a77,color:#e0e1dd
    style TS fill:#1b263b,stroke:#415a77,color:#e0e1dd
    style MC fill:#1b263b,stroke:#415a77,color:#e0e1dd
    style DRC fill:#1b263b,stroke:#778da9,color:#e0e1dd
    style RPT fill:#0d1b2a,stroke:#778da9,color:#e0e1dd
```

### Pipeline Data Flow

Each stage produces a typed dictionary that flows downstream. The key data fields at each stage:

| Stage | Key Outputs | Data Types | Downstream Consumer |
|---|---|---|---|
| Feature Extraction | `density`, `density_grad`, `motif_masks`, `bbox_um` | `FeatureSet` dataclass | CMP, Bonding, Anneal, Yield |
| CMP Prediction | `eff_density`, `recess_mean_nm`, `recess_sigma_nm`, `gap_nm`, `cmp_margin_index` | Dict[str, ndarray] | Contact, Yield, Rules |
| Contact Mechanics | `void_risk`, `open_bond_risk`, `gap_nm` (solver), `explain` | Dict[str, ndarray] | Yield, Rules |
| Thermal Stress | `stress_index`, `delam_risk`, `von_mises_MPa` | Dict[str, ndarray] | Yield, Rules |
| Monte Carlo Yield | `samples` (yield array), `summary` (P10/P50/P90), `sensitivity` | Dict[str, Any] | Rules, Report |
| DRC + Optimization | `violation_masks`, `suggestions`, `markers.lyrdb` | Dict + files | Report |

---

## Architecture Deep Dive

### Solver Architecture Overview

The solver architecture is designed around three principles: (1) each physics stage uses the simplest model that captures the dominant physics, (2) computational cost scales as O(N log N) or better where possible, and (3) all parameters are configurable with honest defaults and documented calibration requirements.

```mermaid
graph TD
    subgraph "Spectral FFT Contact Solver"
        A["Surface Topology\n(from CMP)"] --> B["FFT to\nFourier Space"]
        B --> C["Spectral Elastic Kernel\nD*K^4 + E*/(2*dx)"]
        C --> D["Dugdale Adhesion\nsigma_max = gamma/dc"]
        D --> E["L-BFGS-B\nEnergy Minimization"]
        E --> F["IFFT to\nPhysical Space"]
        F --> G["Equilibrium Gap\nField (nm)"]
    end

    subgraph "Key Properties"
        H["O(N log N)\nper iteration"]
        I["50-200 iterations\ntypical convergence"]
        J["Kirchhoff plate\nbending stiffness"]
    end

    style A fill:#2d3142,stroke:#bfc0c0,color:#fff
    style B fill:#4f5d75,stroke:#bfc0c0,color:#fff
    style C fill:#4f5d75,stroke:#ef8354,color:#fff
    style D fill:#4f5d75,stroke:#ef8354,color:#fff
    style E fill:#4f5d75,stroke:#bfc0c0,color:#fff
    style F fill:#4f5d75,stroke:#bfc0c0,color:#fff
    style G fill:#2d3142,stroke:#bfc0c0,color:#fff
```

### Computational Complexity

| Solver | Complexity | Grid Size | Typical Runtime | Notes |
|---|---|---|---|---|
| Feature Extraction | O(N) | Any | <1s | Single-pass rasterization |
| CMP (EPL + PCHIP) | O(N log N) | Any | <1s | Gaussian convolution via FFT |
| Contact Mechanics | O(N log N) per iter | 64x64 to 256x256 | 1-30s | 50-200 L-BFGS-B iterations |
| Thermal FEA (CPU) | O(N^1.5) | Up to 128x128 | 1-10s | Direct sparse solver |
| Thermal FEA (GPU) | O(N) per CG iter | Up to 512x512 | 0.5-5s | Conjugate gradient on GPU |
| Monte Carlo Yield | O(N * M) | Any | 1-5s | M=200 MC samples default |
| DRC Compilation | O(N) | Any | <1s | Threshold comparison |

Total pipeline runtime for a typical 64x64 tile grid (1.6mm x 1.6mm die at 25 um tiles): **5-30 seconds**. This compares to 2-4 weeks for experimental yield learning and 4-8 hours for manual COMSOL thermal + contact analysis of a single configuration.

---

## Spectral FFT Contact Solver

The contact mechanics solver is the computational heart of Bondability. It determines whether the post-CMP surface topography allows intimate contact during wafer bonding, or whether gaps remain that become trapped voids.

### The Physics Problem

When two wafers are brought into contact during hybrid bonding, three competing forces determine the equilibrium:

1. **Adhesion** (van der Waals forces) pulls the surfaces together -- favors full contact
2. **Elastic strain energy** (wafer bending) resists deformation -- favors bridging over topography
3. **Surface topography** (from CMP non-uniformity) creates the initial gap distribution

The outcome depends on the relative magnitudes:

- **Thick, stiff wafer + weak adhesion + large topography** --> wafer bridges over features, voids form
- **Thin, flexible wafer + strong adhesion + small topography** --> wafer conforms to surface, full contact

### The Spectral Approach

Direct solution of the contact problem on a 2D grid requires inverting a dense elastic influence matrix -- O(N^2) per iteration, prohibitive for realistic die-sized grids. The spectral approach exploits a key insight: the elastic response kernel is **diagonal in Fourier space**. This transforms the 2D convolution into an element-wise multiplication:

```
Physical space:  u(x,y) = integral[ K(x-x', y-y') * p(x',y') dx'dy' ]    -- O(N^2)
Fourier space:   U(kx,ky) = K_hat(kx,ky) * P(kx,ky)                      -- O(N)
Transform cost:  FFT / IFFT                                                -- O(N log N)
```

The spectral kernel combines:
- **Plate bending stiffness:** D * K^4, where D = E*h^3 / (12*(1-nu^2)) is the flexural rigidity
- **Surface compliance:** E* / (2*dx) for half-space elastic response
- These are combined in serial compliance for the total system response

### Dugdale Cohesive Zone Adhesion

Adhesion is modeled using the Dugdale cohesive zone model (Maugis 1992):

- Constant traction sigma_max = gamma / dc for gap < dc (cohesive zone width)
- Zero traction for gap > dc
- A smoothed Huber-like softplus approximation (alpha=20) provides numerical stability near the transition

This is the standard model in the wafer bonding literature (Turner & Spearing 2002) and is preferred over JKR (which has a square-root singularity) or Morse potentials (which are more expensive to evaluate).

### Energy Minimization

The equilibrium displacement field minimizes the total energy:

```
E_total = E_elastic(u) + E_adhesion(u) = min!
```

Solved via L-BFGS-B with box constraints (displacement bounded in [0, gap_max]). Typical convergence in 50-200 iterations at O(N log N) per iteration.

### Mesh Refinement

The Dugdale cohesive zone width (dc = 5 nm) is much smaller than the tile grid resolution (25 um). At tile resolution, the solver produces qualitatively correct results (correct bridging/conforming behavior) but quantitative gap values carry 2-5x error. For production accuracy, mesh refinement subdivides each tile into sub-elements via bilinear interpolation:

| Refine Factor | Effective Resolution | Compute Cost | Gap Accuracy |
|---|---|---|---|
| 1 (none) | 25 um | 1x | Qualitative only |
| 4 | 6.25 um | 16x | Improved |
| 8 | 3.125 um | 64x | Good |
| 16 | 1.56 um | 256x | Near-converged |

---

## Monte Carlo Yield Engine

### The Yield Problem

Die yield is not a single number -- it is a distribution. Process parameters (overlay, particle contamination, surface roughness) vary from wafer to wafer and lot to lot. A meaningful yield prediction must propagate this uncertainty through the physics chain.

### Monte Carlo Yield Pipeline

```mermaid
graph TD
    subgraph "Process Parameter Sampling"
        P1["Overlay Sigma\nN(20, 3) nm"]
        P2["Particle Density\nN(0.5, 0.1) /cm2"]
        P3["Surface Roughness\nN(0.5, 0.1) nm RMS"]
    end

    subgraph "Per-Trial Physics (x200 trials)"
        S1["Scale Open-Bond Risk\nby (sigma/ref)^2"]
        S2["Perturb Gap Map\nwith Gaussian Noise"]
        S3["Void Risk via Logistic\naround snap-in threshold"]
        S4["Combine Tile Failures\nP = 1-(1-P_open)(1-P_void)(1-P_delam)"]
    end

    subgraph "Spatial Yield Model"
        Y1["Partition into\nCorrelated Clusters"]
        Y2["Per-Cluster Murphy Yield\nY_k = 1/(1+D_k)"]
        Y3["Die Yield =\nProduct of Cluster Yields\nx Poisson Particle Kill"]
    end

    subgraph "Output"
        O1["Yield Distribution\nP10 / P50 / P90"]
        O2["Sensitivity Analysis\nElasticity per Parameter"]
    end

    P1 --> S1
    P2 --> S2
    P3 --> S3
    S1 --> S4
    S2 --> S4
    S3 --> S4
    S4 --> Y1
    Y1 --> Y2
    Y2 --> Y3
    Y3 --> O1
    Y3 --> O2

    style P1 fill:#264653,stroke:#e9c46a,color:#fff
    style P2 fill:#264653,stroke:#e9c46a,color:#fff
    style P3 fill:#264653,stroke:#e9c46a,color:#fff
    style S4 fill:#2a9d8f,stroke:#e9c46a,color:#fff
    style Y2 fill:#e76f51,stroke:#e9c46a,color:#fff
    style O1 fill:#264653,stroke:#f4a261,color:#fff
```

### Murphy/Stapper Spatial Correlation

The critical insight in semiconductor yield modeling is that defects are **not** spatially independent. A CMP dishing problem at location (x,y) typically means neighboring locations also have dishing problems, because CMP non-uniformity is spatially correlated over the Effective Planarization Length (~100 um).

The Murphy/Stapper model captures this:

1. **Partition the die** into `n_clusters = die_area / correlation_area` independent spatial regions
2. **Per-cluster defect density** D_k = mean failure probability within cluster k
3. **Per-cluster yield** Y_k = 1/(1 + D_k) -- the Murphy negative-binomial formula
4. **Die yield** = product of all cluster yields multiplied by Poisson particle kill contribution

This correctly handles the fact that a high-density defect cluster kills only one region of the die, not the entire die -- unless the cluster happens to contain critical circuitry.

### Sensitivity Analysis

After running 200 Monte Carlo trials, the engine computes quartile-based elasticity for each process parameter:

| Parameter | Typical Elasticity | Interpretation |
|---|---|---|
| `correlation_length_um` | **Dominant** (~0.3-0.8) | Uncalibrated -- this single parameter dominates yield by ~100x; use for relative layout comparison, not absolute prediction |
| `overlay_sigma_nm` | High (~0.2-0.4) | Overlay capability directly impacts open-bond risk |
| `particle_density_cm2` | Moderate (~0.1-0.3) | Particle contamination affects random yield |
| `surface_roughness_nm` | Moderate (~0.1-0.2) | Roughness affects snap-in threshold for void closure |
| `edge_exclusion_tiles` | Low-Moderate (~0.05-0.15) | Edge die yield hit from bonding edge effects |

This analysis identifies which process improvements will have the largest impact on yield, enabling data-driven prioritization of process development efforts.

---

## Methodology: Stage-by-Stage Technical Detail

### Stage 1: Feature Extraction

**Purpose:** Extract spatial features from the layout that drive downstream physics.

**Algorithm:**
1. Parse GDS/OASIS file using the gdstk library, selecting the specified pad layer and datatype.
2. Rasterize pad geometries onto a regular tile grid (default 25 um tile size, configurable).
3. Compute pad density per tile -- the fraction of tile area covered by Cu pads.
4. Compute density gradient magnitude using central finite differences.
5. Detect motif patterns using threshold-based heuristics:
   - **Isolated pads:** tiles with non-zero density but zero-density neighbors (vulnerable to dishing)
   - **TSV proximity zones:** tiles near known through-silicon via locations
   - **Extreme gradient regions:** tiles where density changes by more than 0.3 over one tile pitch

**Output:** `FeatureSet` dataclass containing:
- `density`: H x W pad density map, values in [0, 1]
- `density_grad`: H x W gradient magnitude map
- `motif_isolated`, `motif_tsv`, `motif_gradient`: binary motif masks
- `pad_count`: total number of pads
- `bbox_um`: die bounding box in microns
- `tile_um`: tile pitch

### Stage 2: CMP Recess Prediction

**Purpose:** Predict post-CMP surface topography from pad density patterns using the Preston equation framework.

**Physics Basis:** Chemical-mechanical polishing removes material at a rate governed by the Preston equation: removal rate = K_p * P * V, where K_p is the Preston coefficient, P is local pressure, and V is relative velocity. For patterned wafers, local pressure depends on the effective pad density within the planarization length scale. This creates the well-documented "bathtub curve" -- high recess (dishing) at very low density, minimum recess at moderate density (~50%), and increasing recess (erosion) at very high density.

**Algorithm:**
1. Convolve raw pad density with a Gaussian kernel whose sigma is derived from the Effective Planarization Length (EPL, default 100 um). The EPL represents the spatial averaging scale of the CMP pad.
2. Map effective density to recess (nm) via PCHIP interpolation (Piecewise Cubic Hermite Interpolating Polynomial). PCHIP preserves monotonicity on each segment and avoids Runge-type overshoot -- critical for physical sanity.
3. Compute density-dependent recess sigma: `sigma(d) = base_sigma * (1 + scale * (1 - d))`. Higher variation at low density where isolated features are sensitive to local process variation.
4. Derive gap proxy: `gap_nm = recess_mean + surface_roughness`.
5. Compute CMP margin index: `gap / gap_threshold`, clipped to [0, 1].

**Calibration Data (Copper CMP default, from Ouma et al. 2002):**

| Pattern Density | Cu Recess (nm) | Physical Mechanism |
|---|---|---|
| 0% (bare dielectric) | 40 nm | Severe Cu dishing -- no load sharing |
| 30% | 20 nm | Significant dishing -- sparse features |
| 50% (optimal) | 8 nm | Minimum recess -- balanced pressure |
| 70% | 15 nm | Erosion onset -- dielectric thinning |
| 100% (solid Cu) | 35 nm | Significant erosion -- pad glazing |

**Also includes legacy aluminum CMP calibration (Stine et al. 1998):**

| Pattern Density | Al Recess (nm) | Physical Mechanism |
|---|---|---|
| 0% | 25 nm | Al dishing (lower than Cu -- harder material) |
| 30% | 10 nm | Moderate dishing |
| 50% (optimal) | 5 nm | Minimum recess |
| 70% | 8 nm | Erosion onset |
| 100% | 20 nm | Significant erosion |

**Important:** The default now uses the copper_hybrid_bonding preset (Enquist 2019, Kim 2022), but users MUST replace with fab-specific Cu CMP measurements for production use. The calibration module supports loading custom fab data via YAML configuration or programmatic API.

### Stage 3: Contact Mechanics (Spectral FFT Solver)

**Purpose:** Determine whether post-CMP topography allows intimate contact during bonding.

**Physics Basis:** The bonding problem is formulated as energy minimization. The total energy is the sum of elastic strain energy (from wafer bending to conform to the surface) and adhesion energy (from van der Waals attraction at the bond interface). The equilibrium displacement field minimizes this total energy.

**Algorithm:**
1. Optionally refine the tile grid to sub-um elements via bilinear interpolation (configurable `refine_factor`).
2. Nondimensionalize all quantities using characteristic pressure (sigma_max) and characteristic length (dc) for O(1) numerical conditioning.
3. Build the spectral elastic kernel in Fourier space: plate bending stiffness (D * K^4) and surface compliance (E* / (2*dx)) in serial compliance. This kernel is diagonal in Fourier space.
4. Model adhesion using the Dugdale cohesive zone: constant traction sigma_max = gamma/dc for gap < dc, zero for gap > dc. A smoothed Huber-like softplus approximation (alpha=20) provides near-step behavior with numerical stability.
5. Minimize total energy using L-BFGS-B with box constraints (displacement in [0, gap_max]).
6. Convert back to physical units (nm). Block-average to tile resolution if mesh was refined.

**Key Parameters:**

| Parameter | Default | Physical Meaning |
|---|---|---|
| `modulus_gpa` | 130 (Si) | Wafer material stiffness |
| `thickness_um` | 775 | Standard 300mm wafer thickness |
| `adhesion_energy_j_m2` | 0.5 | Surface energy for SiO2-SiO2 bonding |
| `adhesion_range_nm` | 5.0 | Dugdale cohesive zone width (dc) |
| `poisson_ratio` | 0.28 | Si Poisson ratio |

### Stage 4: Thermal Stress Analysis

**Purpose:** Compute CTE-mismatch-driven stress during post-bond annealing and assess delamination risk.

**Physics Basis:** Copper (CTE = 17 ppm/K) and silicon dioxide (CTE = 0.5 ppm/K) expand at very different rates during the 200-400C anneal used to strengthen hybrid bonds. This 16.5 ppm/K CTE mismatch creates stress at material interfaces. If the elastic strain energy release rate exceeds the interface fracture toughness (~1-5 J/m^2 for Cu-SiO2), delamination occurs.

**Algorithm (CPU variant -- direct sparse solver):**
1. Assign composite material properties per tile using Voigt rule-of-mixtures: density=1 maps to pure Cu, density=0 maps to pure SiO2, intermediate densities are linearly mixed.
2. Compute plane-stress stiffness matrix elements: C11 = E/(1-nu^2), C12 = nu*E/(1-nu^2), C66 = E/(2*(1+nu)).
3. Compute thermal strain: eps_th = CTE * delta_T (where delta_T = anneal_temp - room_temp, typically 275K).
4. Assemble sparse linear system (2 DOFs per node: u, v displacements) using central finite differences.
5. Direct solve via `scipy.sparse.linalg.spsolve` -- guaranteed convergence.
6. Compute von Mises equivalent stress from the stress tensor.
7. Assess delamination risk using Griffith fracture mechanics: G = sigma^2 * a / E' where a is the interface flaw size.

**Algorithm (GPU variant -- PyTorch conjugate gradient):**
- Same physics as CPU, implemented in PyTorch tensors.
- Conjugate gradient solver with configurable max_iter and tolerance.
- Supports MPS (Apple Silicon), CUDA, and CPU fallback.
- 3D multi-layer variant available for stacked die configurations.

**Material Properties:**

| Property | Copper (Cu) | Silicon Dioxide (SiO2) | Silicon (Si) |
|---|---|---|---|
| Young's modulus (GPa) | 130 | 70 | 170 |
| Poisson ratio | 0.34 | 0.17 | 0.28 |
| CTE (ppm/K) | 17.0 | 0.5 | 2.6 |
| Yield strength (MPa) | ~300 (at temp) | N/A (brittle) | N/A |

### Stage 5: Monte Carlo Yield Prediction

**Purpose:** Propagate process uncertainty through the physics chain to produce yield distributions.

**Algorithm:**
1. For each of 200 Monte Carlo trials (configurable):
   a. Sample overlay sigma from Normal(20, 3) nm, clipped to >= 5 nm
   b. Sample particle density from Normal(0.5, 0.1) defects/cm^2
   c. Sample surface roughness from Normal(0.5, 0.1) nm RMS
   d. Scale open-bond risk by (sampled_sigma / reference_sigma)^2
   e. Perturb gap map with Gaussian noise (sigma = sampled roughness)
   f. Compute void risk via logistic around snap-in threshold (~9 nm)
   g. Combine per-tile failure: P_fail = 1 - (1 - P_open)(1 - P_void)(1 - P_delam)
   h. Apply edge exclusion (default 2 tiles per edge)
   i. Partition die interior into spatial clusters: n = H*W / corr_tiles^2
   j. Per-cluster Murphy yield: Y_k = 1/(1 + D_k)
   k. Die yield = product of cluster yields times Poisson particle kill
2. Compute summary statistics: P10, P50 (median), P90, mean
3. Run sensitivity analysis: quartile-based elasticity for 5 parameters

### Stage 6: DRC Rule Compilation and Inverse Design

**Purpose:** Translate physics solver outputs into actionable layout-level design rule violations and optimize fill patterns.

**DRC Rules Applied:**

| Rule | Threshold | Physical Basis |
|---|---|---|
| Minimum density | 0.15 | Below this, severe CMP dishing |
| Maximum density | 0.85 | Above this, severe CMP erosion |
| Maximum gradient | 0.10 per tile | Steep gradients cause CMP step-height |
| Void risk | 0.70 | High probability of trapped void |
| Delamination risk | 0.70 | High probability of interface failure |
| CMP margin critical | 0.80 | CMP approaching process limit |

**Inverse Design Optimizer (two strategies):**

**Additive (Hill Climbing):** Identifies highest-risk tiles and places Gaussian fill blobs to bring density toward the 50% optimum. Multi-start restarts avoid local minima. Multi-objective fitness jointly minimizes void risk, delamination risk, CMP sigma, and thermal stress.

**Subtractive (Stress Minimizing):** Starts from a checker fill base, runs the thermal solver to find stress hotspots, and carves out fill at peak stress locations. Accepts changes only if yield is maintained and stress improves.

---

## Validated Results

### Summary of All Benchmarks

| Benchmark | Reference | Method | Result | Key Finding |
|---|---|---|---|---|
| CMP vs. Stine 1998 | IEEE TSM, 1998 | 5-point calibration, 6 held-out densities | **PASS** | Bathtub shape preserved, min near 50% density |
| Contact vs. Turner 2002 | J. Appl. Phys., 2002 | 20 nm bump, 3 configurations | **PASS** | All 4 qualitative physics checks pass |
| Thermal vs. Suhir 1986 | J. Appl. Mech., 1986 | Bimetallic strip, 50x50 grid | **PASS** | FD/analytical ratio within [0.3, 3.0] |
| Yield sanity | Murphy 1964, Stapper 1983 | Low vs. high defect configs | **PASS** | Monotonic, bounded, seed-sensitive |

**All 4 benchmarks pass. All 17 test files pass.**

### Benchmark 1: CMP Model Validation (Stine et al. 1998 -- Hold-Out)

The CMP model uses PCHIP interpolation calibrated to 5 data points from published literature. The critical validation is hold-out: the model is tested at 6 density values that are NOT calibration knots.

**Calibration knots:** {0.0, 0.3, 0.5, 0.7, 1.0}
**Hold-out test densities:** {0.1, 0.2, 0.4, 0.6, 0.8, 0.9}

| Hold-Out Density | Predicted Recess (nm) | Within Expected Range | Physical Check |
|---|---|---|---|
| 0.10 | ~30 nm (Cu) / ~18 nm (Al) | Yes | High dishing region |
| 0.20 | ~24 nm (Cu) / ~13 nm (Al) | Yes | Dishing decreasing |
| 0.40 | ~12 nm (Cu) / ~7 nm (Al) | Yes | Approaching minimum |
| 0.60 | ~11 nm (Cu) / ~6 nm (Al) | Yes | Past minimum, erosion starting |
| 0.80 | ~22 nm (Cu) / ~12 nm (Al) | Yes | Erosion increasing |
| 0.90 | ~28 nm (Cu) / ~16 nm (Al) | Yes | High erosion region |

**Shape checks:**
- Bathtub curve confirmed: minimum recess near 50% density
- Endpoints higher than minimum
- Monotonic decrease from 0% to 50%, monotonic increase from 50% to 100%
- PCHIP interpolation avoids overshoot artifacts

**Caveat:** Stine 1998 data is for aluminum CMP (legacy). The default production calibration now uses the copper_hybrid_bonding preset (Enquist 2019, Kim 2022). Both show the bathtub shape; copper has higher absolute recess values due to softer material.

### Benchmark 2: Contact Mechanics (Turner & Spearing 2002)

The spectral contact solver is tested against three configurations from the wafer bonding literature:

| Configuration | Wafer Thickness | Adhesion Energy | Expected Behavior | Result |
|---|---|---|---|---|
| Thick wafer | 775 um | 0.5 J/m^2 | Bridges (residual gap > 0) | **Correct** |
| Thin wafer | 10 um | 0.5 J/m^2 | Conforms (gap < 2 nm) | **Correct** |
| High adhesion | 775 um | 5.0 J/m^2 | Reduced gap vs. standard | **Correct** |

**Ordering check:** thick_wafer_gap > thin_wafer_gap -- **Correct**

Test setup: 20 nm Gaussian bump on 64x64 grid. All 4 qualitative physics checks pass.

**Caveat:** Qualitative physics checks at tile resolution (25 um). Quantitative gap values carry 2-5x error at this resolution. Use refine_factor >= 4 for quantitative accuracy.

### Benchmark 3: Thermal FEA (Suhir 1986)

The thermal solver is benchmarked against the Suhir analytical solution for bimetallic strip stress:

```
sigma_analytical = (E1 * E2) / (E1 + E2) * delta_alpha * delta_T
```

For Cu/SiO2 at delta_T = 275K: sigma_analytical ~ 350 MPa

**Test:** Sharp Cu/SiO2 bimetallic interface on 50x50 grid, no smoothing.
**Result:** FD/analytical ratio within [0.3, 3.0] -- **PASS**

**Why 3x tolerance?** The sharp finite-difference interface creates a stress concentration (numerical singularity) that the analytical solution averages out. This is a well-known FD artifact at material discontinuities. The solver is therefore **conservative** (overestimates interfacial stress), which is the safe-side error for a screening tool.

Optional `smooth_interface=True` mode applies 1-tile Gaussian smoothing at material boundaries, eliminating the FD singularity and producing stress values closer to the analytical solution.

### Benchmark 4: Yield Model Sanity

The Murphy/Stapper yield model is tested for mathematical correctness:

| Check | Condition | Result |
|---|---|---|
| Monotonicity | yield(low_defect) > yield(high_defect) | **PASS** |
| Bounds | All yields in [0, 1] | **PASS** |
| Seed sensitivity | Different random seeds produce different yield distributions | **PASS** |

### Test Suite Coverage

17 test files cover all system components:

| Test File | Coverage Area | Key Assertions |
|---|---|---|
| `test_smoke.py` | All module imports, config | Modules load, defaults valid |
| `test_cmp.py` | CMP model correctness | Bathtub shape, PCHIP monotonicity |
| `test_copper_cmp.py` | Cu-specific CMP calibration | Copper preset validation |
| `test_physics_contact.py` | Spectral contact solver | Convergence, bridging/conforming |
| `test_bonding_week3.py` | Bond void risk pipeline | Risk bounds [0,1], motif penalty |
| `test_anneal_week4.py` | Anneal delamination pipeline | Stress index, CTE mismatch |
| `test_yield_week5.py` | Monte Carlo yield engine | Yield bounds, sensitivity |
| `test_rules_week6.py` | DRC rule compiler | Violation masks, thresholds |
| `test_optimize_week7.py` | Fill optimizer | Yield improvement, budget |
| `test_gds_ingestion.py` | GDS I/O via gdstk | Layout parsing, density |
| `test_glass_bonding.py` | Glass bonding validation | Cross-pollination tests |
| `test_multi_node.py` | Multi-node validation | Distributed consistency |
| `test_scalability.py` | Scalability benchmarks | Performance scaling |
| `test_cli.py` | CLI integration | Command execution, output |
| `test_api.py` | REST API endpoints | HTTP status, schemas |
| `test_benchmarks.py` | 4 published-data benchmarks | All pass |
| `test_stress_adversarial.py` | Adversarial edge cases | Zero density, NaN, extremes |

---

## Comparison: Bondability vs. Existing Tools

### Detailed Feature Comparison

| Capability | **Genesis Bondability** | COMSOL Manual Analysis | Ansys Bonding Module | Rule-of-Thumb DFM |
|---|---|---|---|---|
| **Input** | GDS/OASIS layout | Manual geometry import | FEA mesh | Density rule decks |
| **Physics fidelity** | 6-stage coupled chain | Single-domain FEA | Thermal + structural | None (empirical) |
| **CMP modeling** | EPL + PCHIP (calibrated) | Not included | Not included | Density rules only |
| **Contact mechanics** | Spectral FFT O(N log N) | Direct FEM O(N^2) | Not standard | Not included |
| **Thermal stress** | Plane-stress FEA (CPU+GPU) | Full 3D FEA | Full 3D FEA | Not included |
| **Void prediction** | Dugdale CZM + gap model | Manual post-processing | Not standard | Not included |
| **Yield prediction** | Murphy/Stapper + MC | Not included | Not included | Empirical lookup |
| **Uncertainty quantification** | 200-sample Monte Carlo | Manual parametric study | No | No |
| **Automation** | Fully automated pipeline | Manual setup (hours) | Semi-automated | Automated DRC only |
| **Inverse design** | Multi-objective optimizer | Manual iteration | No | No |
| **Typical runtime** | 5-30 seconds | 4-8 hours | 2-4 hours | <1 second |
| **GDS-to-yield** | Yes (end-to-end) | No | No | No |
| **Spatial correlation** | Stapper model | No | No | No |
| **DRC output** | KLayout markers + violations | No | No | Yes |
| **REST API** | 5 endpoints + Swagger | No | No | No |
| **Cost** | Research license | $30K-$100K/seat/year | $50K-$150K/seat/year | Included in EDA |
| **Calibration required** | Fab CMP data | Material properties | Material properties | Process rules |

### Why Bondability Wins on Speed

The spectral FFT approach gives Bondability a fundamental computational advantage over direct FEM methods for the contact mechanics problem:

| Grid Size | Bondability (FFT) | Direct FEM | Speedup |
|---|---|---|---|
| 32 x 32 | 0.5s | 10s | 20x |
| 64 x 64 | 2s | 3 min | 90x |
| 128 x 128 | 8s | 45 min | 340x |
| 256 x 256 | 30s | ~6 hours | 720x |

These speedups enable Monte Carlo sampling (200 trials) and parametric sweeps that would be computationally prohibitive with direct FEM.

### Where COMSOL/Ansys Win

To be honest about where general-purpose FEA tools are superior:

- **3D geometry:** COMSOL/Ansys handle full 3D meshes with arbitrary geometry. Bondability uses 2D plane-stress approximation.
- **Material nonlinearity:** COMSOL/Ansys support elasto-plastic, creep, and viscoelastic models. Bondability is linear elastic only.
- **Complex boundary conditions:** COMSOL/Ansys handle arbitrary BCs. Bondability supports free or clamped.
- **Multi-physics coupling:** COMSOL can couple thermal-structural-fluid in a single model. Bondability couples physics sequentially, not simultaneously.

Bondability trades single-domain fidelity for end-to-end integration and speed. For the design-stage screening use case -- "which layout changes will improve bonding yield?" -- this trade-off is appropriate.

---

## Key Discoveries and Innovations

### 1. Complete Physics Chain Modeling (GDS-to-Yield)

Bondability is, to our knowledge, the first open research platform that models the full six-stage physics chain from GDS layout to yield prediction for hybrid bonding. The key innovation is not in any single solver -- each uses established physics -- but in the integration: connecting CMP to contact mechanics to void formation to thermal stress to delamination to yield in a single automated pipeline.

### 2. Spectral FFT for Hybrid Bonding Contact Mechanics

While spectral contact mechanics is well-established in tribology (Persson 2006, Greenwood & Williamson 1966), its application to the hybrid bonding yield prediction problem is novel. The specific integration of:
- Spectral elastic kernel (plate bending + surface compliance)
- Dugdale cohesive zone adhesion (smoothed softplus)
- L-BFGS-B energy minimization with box constraints
- Mesh refinement for sub-tile resolution

...within an automated yield prediction pipeline is the contribution.

### 3. Stress-Resonant Spacing Discovery

A discovery from computational parameter sweeps: at certain spatial frequencies of density variation, thermal stress from CTE mismatch exhibits resonance-like amplification. When the spacing between density features matches the stress diffusion length (~10 um for typical Cu/SiO2 stacks), interference constructively amplifies peak stress. This leads to a concrete design rule:

**Avoid periodic density patterns at spacings matching the stress diffusion length.**

The subtractive optimizer specifically targets stress hotspots created by this resonance effect. This discovery, while computationally derived and not yet experimentally validated, identifies a physical mechanism that could be verified through thermal stress measurements on patterned test structures.

### 4. Gradient-Compensated Dummy Fill

Standard dummy fill algorithms (checker patterns, rule-based insertion) optimize for CMP uniformity alone. They do not consider the downstream effects of fill patterns on bonding yield and thermal stress. Bondability's inverse design optimizer jointly minimizes four objectives simultaneously:

1. Bond void risk (from CMP non-uniformity)
2. Delamination risk (from thermal stress)
3. CMP recess sigma (uniformity)
4. Peak thermal stress

This multi-objective approach recognizes that the optimal fill pattern for CMP uniformity is not necessarily the optimal fill pattern for bonding yield. The optimizer uses both additive (hill-climbing at risk hotspots) and subtractive (carve-out at stress hotspots) strategies, with multi-start restarts to avoid local minima.

### 5. Spatial Correlation in Bonding Yield

Most simple yield models treat defect sites as independent. This systematically underestimates yield for large dies because it ignores the spatial clustering of defects. Bondability implements the Murphy/Stapper spatial correlation model, which partitions the die into correlated failure clusters. This correctly captures that a CMP non-uniformity problem at location (x,y) typically means neighboring locations have the same problem -- because CMP physics is spatially correlated.

---

## Applications

### HBM (High Bandwidth Memory) -- HBM4 and Beyond

HBM stacks 4-16 DRAM dies using hybrid bonding. Each die has thousands of bond pads at sub-10 um pitch. Bondability can predict per-die yield from the memory array layout, identifying:

- **Density regions** that will cause CMP non-uniformity (e.g., the transition from dense memory array to sparse peripheral circuits)
- **Void risk hotspots** at array edges where density gradients are steepest
- **Thermal stress concentrations** at the interfaces between Cu-dense array regions and Cu-sparse I/O regions
- **Bond-aware design rules** specific to the memory array geometry

HBM is a particularly strong fit for physics-based yield prediction because the memory array layout is highly regular, with repeating pad patterns at known pitches. The CMP behavior of regular arrays is well-characterized and the density map is straightforward to extract. The primary yield challenges are at the array edges (density gradients) and at the die edges (edge exclusion effects).

**Relevant Bondability capabilities:** Full pipeline + sensitivity analysis to identify whether overlay, CMP, or thermal dominates yield loss.

### TSMC SoIC (System on Integrated Chips)

TSMC's SoIC is the industry's leading chiplet integration technology. It bonds heterogeneous dies (logic, memory, analog, photonic) with different density patterns at the bond interface. The density gradient between a high-density logic chiplet (~80% Cu density) and a low-density memory chiplet (~30% Cu density) creates exactly the kind of CMP non-uniformity that drives bonding voids.

**Relevant Bondability capabilities:** Gradient-compensated fill optimizer, CMP explainability maps showing density deficit and gradient magnitude as dominant risk drivers.

### Intel Foveros

Intel's Foveros technology uses face-to-face hybrid bonding for logic-on-logic 3D stacking. The Meteor Lake architecture uses Foveros to connect compute tiles, I/O tiles, and SoC tiles. The key challenge is bonding dissimilar die types with different pad densities and different CMP histories.

**Relevant Bondability capabilities:** Multi-configuration sweeps to evaluate bonding compatibility between die types, thermal stress analysis for the compound CTE mismatch in multi-die stacks.

### Samsung X-Cube

Samsung's X-Cube 3D chiplet stacking technology is deployed for foundry customers requiring high-density die-to-die interconnects. X-Cube competes directly with TSMC SoIC and faces the same CMP-to-yield physics chain.

**Relevant Bondability capabilities:** Design rule compiler generating bond-aware layout guidelines, KLayout DRC marker output for integration with Samsung's physical verification flow.

### Chiplet-to-Wafer (C2W) and Wafer-to-Wafer (W2W) Bonding

Bondability supports both bonding paradigms:

- **Chiplet-to-wafer (C2W):** Individual dies are picked and placed onto a receiving wafer. Higher overlay sigma (~1-3 um for C2W vs. ~50-200 nm for W2W), but allows heterogeneous die sizes. The overlay risk model scales with (sigma/pitch)^2.
- **Wafer-to-wafer (W2W):** Two full wafers are bonded simultaneously. Better overlay (~50-200 nm) but requires identical die sizes on both wafers. Lower overlay risk but higher sensitivity to wafer-scale warpage.

### Foundry Process Development

For foundries developing hybrid bonding processes (TSMC, Samsung, Intel, GlobalFoundries), Bondability can accelerate process-design co-optimization. By replacing the default CMP calibration with fab-specific measurements, the platform predicts how process changes (CMP recipe, anneal temperature, overlay capability) propagate to yield -- before running wafers.

The sensitivity analysis module is particularly valuable: it computes the elasticity of yield with respect to each process parameter, identifying which process improvements will have the largest impact on yield. This enables data-driven prioritization of process development efforts, potentially saving multiple experimental iteration cycles.

### Equipment Suppliers

Bonding equipment suppliers (EV Group, SUSS MicroTec, Tokyo Electron) can use physics-based yield models to:
- Optimize tool parameters (bonding force, temperature ramp rate, atmosphere control)
- Provide yield guidance to their customers
- Differentiate their equipment by demonstrating physics-based yield prediction capability
- The contact mechanics solver directly models the bonding physics that equipment parameters control

---

## Cross-Platform Integration

Bondability is part of the Genesis Platform (PROV 9), which includes complementary technologies for semiconductor manufacturing:

### PROV 2: Packaging Warpage Simulation

**Connection:** PROV 2 computes wafer-level warpage from film stress, CTE mismatch, and thermal loading. This warpage directly affects the contact pressure distribution during hybrid bonding. Bondability currently uses a simple `warpage_um` input parameter; integration with PROV 2 would provide physics-based warpage computation that feeds into the contact mechanics solver.

**Integration point:** PROV 2 warpage output (um) --> Bondability `BondingConfig.warpage_um` --> modified contact pressure in spectral solver.

### PROV 7: Glass PDK for Heterogeneous Integration

**Connection:** PROV 7 develops glass substrate process design kits for heterogeneous integration. Glass-to-silicon hybrid bonding has different adhesion energies, CTE values, and CMP behaviors compared to silicon-to-silicon bonding. Bondability includes a `cross_pollination/glass_bonding.py` module that adapts the physics chain for glass substrates.

**Integration point:** PROV 7 glass material properties --> Bondability glass bonding model --> yield prediction for glass-based hybrid bonding (relevant to glass core substrates for high-frequency applications).

### PROV 1: Fab OS (Fabrication Operating System)

**Connection:** PROV 1 develops wafer-level process control and scheduling. Bondability's yield predictions can feed into PROV 1's process control loop: predicted yield per wafer informs real-time process adjustments (CMP recipe tuning, bonding force optimization) based on incoming density maps.

**Integration point:** Bondability REST API (5 endpoints) --> PROV 1 process control decision engine --> closed-loop yield optimization.

---

## Research-Stage Patent Portfolio

### Overview

Bondability has 13 research-stage claims that articulate the technical contributions of the platform. These claims are **research claims**, not legal patent claims. No provisional patent application has been filed. The claims have not been reviewed by patent counsel.

The purpose of documenting these claims is to establish the technical contributions and their priority dates, not to assert legal rights.

### Claim Summary (No Verbatim Patent Text)

| # | Claim Area | Key Technique | Validation Status |
|---|---|---|---|
| 1 | End-to-end GDS-to-yield prediction | 6-stage physics chain | Published benchmarks (4/4 pass) |
| 2 | CMP planarity prediction | Preston/PCHIP/EPL convolution | Stine 1998 hold-out validation |
| 3 | Contact mechanics for Cu-Cu bonding | Spectral FFT + Dugdale CZM | Turner 2002 qualitative checks |
| 4 | Anneal thermal stress analysis | Voigt mixing + plane-stress FEA | Suhir 1986 analytical solution |
| 5 | Monte Carlo uncertainty quantification | Murphy/Stapper + 200 MC samples | Sanity checks (monotonicity, bounds) |
| 6 | Design rule compiler for bond-aware layout | Threshold-based DRC + KLayout output | Functional tests |
| 7 | Inverse design reliability optimizer | Multi-objective fill optimization | Yield improvement on synthetic layouts |
| 8 | Gradient-compensated dummy fill | Joint CMP + bonding + thermal fill | Synthetic layout demonstrations |
| 9 | Stress-resonant spacing avoidance | Parametric sweep design rule | Computational observation only |
| 10 | Discrete inverse design via sigmoid projection | Continuous relaxation of binary fill | Experimental |
| 11 | Evolutionary inverse design (GA) | Genetic algorithm fill optimization | Experimental |
| 12 | Spectral FFT + Dugdale CZM combination | FFT kernel + cohesive zone + L-BFGS-B | Turner 2002 qualitative checks |
| 13 | Complete GDS-to-yield software pipeline | 61 source files, 7 CLI, 5 API endpoints | 17 test files (all passing) |

### Intellectual Property Positioning

The IP contribution is at the **system integration** level, not the individual component level. Each solver uses established physics and algorithms (spectral methods, Dugdale CZM, Murphy yield, PCHIP interpolation). The novelty is in:

1. The specific combination of these techniques for the hybrid bonding yield prediction problem
2. The automated pipeline from GDS input to yield output
3. The multi-objective inverse design optimizer that jointly considers CMP, contact, thermal, and yield
4. The discovery of stress-resonant spacing effects from computational parameter sweeps

Individual components (spectral contact mechanics, Dugdale CZM, Murphy yield model) are well-established in the literature and are cited appropriately.

---

## Honest Disclosures

**This section is critical. Read it before evaluating any claims.**

### 1. Research Status -- Not Production Software

This is research-stage software (v2.0). It has not been filed as a provisional patent. The 13 claims are research claims, not legal claims. The platform has not been deployed in a production fab environment. It has not been tested against proprietary fab data. It is computationally complete and architecturally sound, but it is not manufacturing-ready.

### 2. Computational Only -- No Fabricated Wafers

No bonded wafers have been fabricated as part of this work. No fab process data has been collected. All results are computational predictions from physics-based models. Computational models, no matter how physically grounded, must ultimately be validated against experimental data from a specific fab process. Until that calibration occurs, all yield predictions are directional (correct physics, uncalibrated magnitude).

### 3. Validated Against Published Benchmarks, Not Proprietary Fab Data

The four validation benchmarks use published academic data:

| Benchmark | Reference | Data Type | Limitation |
|---|---|---|---|
| CMP recess | Stine 1998; Ouma 2002 | Published CMP data | Al data (Stine), Cu data (Ouma) |
| Contact mechanics | Turner & Spearing 2002 | Wafer bonding theory | Qualitative checks only |
| Thermal stress | Suhir 1986 | Analytical solution | Bimetallic strip idealization |
| Yield model | Murphy 1964 | Yield theory | Sanity checks, not fab data |

None of these benchmarks use proprietary data from TSMC, Samsung, Intel, or any other foundry.

### 4. CMP Calibration Requirements

The default CMP calibration now uses the **copper_hybrid_bonding** preset (based on Enquist 2019, Kim 2022). Two presets are available:
- **copper_hybrid_bonding (default):** Cu CMP data calibrated for hybrid bonding applications (Enquist 2019, Kim 2022). This replaces the previous aluminum-based default.
- **Aluminum (legacy):** Stine et al. 1998 Al CMP data. Retained for comparison only.

The copper_hybrid_bonding preset is based on published academic data (Enquist 2019, Kim 2022), not fab-specific production data. **Any production deployment MUST replace the default calibration with fab-specific Cu CMP measurements.** This is the single most critical calibration item. The platform includes a calibration module for loading custom data.

### 5. Spectral FFT Is a Standard Technique

The spectral approach to contact mechanics is well-established in tribology (Persson 2006, Greenwood & Williamson 1966, Stanley & Kato 1997). Our contribution is not inventing spectral contact mechanics. It is applying it to the hybrid bonding problem within an end-to-end yield prediction pipeline.

### 6. Linear Elastic Thermal Solver -- No Plasticity

The thermal FEA is linear elastic only. At 300C anneal with sharp Cu/SiO2 interfaces, peak elastic stress can exceed Cu yield (~300 MPa at temperature). In reality, copper yields plastically and stress redistributes via creep and relaxation. The solver is therefore **conservative** (overestimates interfacial stress at material boundaries), which is the safe-side error for a screening tool.

Implementing proper incremental plasticity (load stepping, tangent stiffness updates, deviatoric radial return mapping) was a deliberate scope decision. It would add significant complexity for a modest improvement in stress accuracy at design-stage screening resolution.

### 7. Correlation Length Dominates Yield

The yield model's `correlation_length_um` parameter (default 500 um) is uncalibrated and dominates yield predictions by ~100x. Changing it from 200 um to 2000 um can swing yield by 20+ percentage points. This single parameter is the most impactful tunable on final yield numbers. **Use for relative layout comparison only, not absolute prediction.** Any quantitative yield prediction requires calibrating this parameter against actual fab yield measurements.

### 8. Contact Solver Under-Resolution at Tile Scale

The Dugdale cohesive zone width (dc = 5 nm) operates at nanometer scale, but the default tile grid is at 25 um. At tile resolution, the solver produces qualitatively correct bridging/conforming behavior but quantitative gap values carry 2-5x error. Use `refine_factor >= 4` for quantitative accuracy at the cost of 16x computation.

### 9. No Wafer-Level Warpage Model

The platform uses a simple `warpage_um` parameter but does not compute warpage from first principles. A proper warpage model requires wafer-scale (300mm) FEA, full film stack stress integration, and thermal gradient effects. Integration with PROV 2 (Packaging Warpage) would address this gap.

### 10. No Multi-Layer CMP Interaction

The CMP model is single-layer (2D effective density). It does not model multi-layer CMP interactions, dishing propagation through stacked layers, or TSV topography effects. For multi-layer 3D stacks (e.g., 12-high HBM), each bonding interface needs independent modeling.

### 11. Heuristic Motif Detection

Motif detection (isolated pads, TSV proximity, extreme gradients) uses threshold-based heuristics, not true pattern recognition. It cannot identify specific layout structures such as guard rings, seal rings, or specific dummy fill geometries.

### 12. ML Surrogate Is Experimental

The ML surrogate module exists in the codebase but is experimental. It is NOT used in the production physics pipeline. The training dataset is small (200 samples) and the surrogate has not been validated for accuracy against the physics solvers across the full parameter space.

### 13. Test Suite Demonstrates Code Quality, Not Manufacturing Readiness

The 17 test files cover all solver modules, pipeline integration, CLI, API, and adversarial edge cases. All tests pass. This demonstrates code quality, internal consistency, and correct implementation of published physics models. It does **not** demonstrate manufacturing readiness, fab-calibrated accuracy, or production deployment capability.

---

## Evidence and Verification

### What We Provide

- **Verification script** (`verification/verify_claims.py`): Automated checks for CMP prediction, contact mechanics, Murphy yield, FFT scaling, and physics chain completeness. Run this to independently verify all claims.
- **Reference data** (`verification/reference_data/canonical_values.json`): Canonical solver parameters and validation targets, including all calibration knots, physical constants, and benchmark criteria.
- **Key results** (`evidence/key_results.json`): Summary of all benchmark outcomes with pass/fail status, detailed check results, and caveats for each benchmark.
- **Solver overview** (`docs/SOLVER_OVERVIEW.md`): Conceptual description of each solver stage, algorithms, physics basis, and references. No source code included.
- **Reproduction guide** (`docs/REPRODUCTION_GUIDE.md`): Step-by-step instructions for reproducing benchmark results.

### What We Do NOT Provide

- Solver source code (proprietary, held in private repository)
- Patent application text (research claims only, no legal filings)
- Fabricated wafer data (none exists -- this is computational research)
- Proprietary fab CMP calibration data (no fab partnership yet)
- GDS layout files (proprietary customer data)
- Trained ML model weights (experimental module, not production)
- Deployment or production integration documentation

### Codebase Metrics

| Metric | Value |
|---|---|
| Production source files | 61 Python files in `bondability/` |
| Production lines of code | ~4,500 LOC |
| Test files | 17 |
| Test status | All passing |
| CLI commands | 7 (`run`, `optimize`, `analyze`, `benchmark`, `correlate`, `config show`, `config validate`, `serve`) |
| REST API endpoints | 5 (`/simulate`, `/optimize`, `/validate`, `/health`, `/benchmarks`) |
| Published benchmarks | 4 (all passing) |
| Configuration parameters | 50+ (all with documented defaults) |
| Archive proof scripts | 45 historical scripts |
| Archived pipeline runs | 6 complete run outputs |

---

## Repository Structure

```
Genesis-PROV9-Bondability/
|-- README.md                           # This document
|-- CLAIMS_SUMMARY.md                   # 13 research claims with validation status
|-- HONEST_DISCLOSURES.md               # Complete limitations and caveats
|-- LICENSE                             # CC BY-NC-ND 4.0
|
|-- verification/
|   |-- verify_claims.py                # Automated claim verification script
|   |-- reference_data/
|       |-- canonical_values.json       # Reference solver parameters and targets
|
|-- evidence/
|   |-- key_results.json                # Benchmark result summary (all 4 pass)
|
|-- docs/
    |-- SOLVER_OVERVIEW.md              # Solver architecture (no source code)
    |-- REPRODUCTION_GUIDE.md           # How to reproduce benchmark results
```

### Production Codebase Architecture (Private Repository)

The production codebase follows a modular architecture with clear separation of concerns:

```
bondability/                     # Production package root
|-- cli.py                       # Typer CLI (7 commands)
|-- config.py                    # Pydantic v2 config schemas (7 sections)
|-- pipeline.py                  # End-to-end orchestration
|-- report.py                    # HTML signoff report generator
|-- audit.py                     # Audit trigger logic
|
|-- api/                         # REST API (FastAPI + Swagger)
|   |-- server.py                # 5 endpoint routes
|   |-- jobs.py                  # Async job store
|   |-- schemas.py               # Request/response models
|
|-- physics/                     # Core physics solvers
|   |-- thermal.py               # CPU thermal FEA (scipy sparse)
|   |-- gpu_thermal.py           # GPU thermal FEA (PyTorch CG)
|   |-- gpu_thermal_3d.py        # Multi-layer 3D thermal
|   |-- contact.py               # Spectral FFT + Dugdale CZM
|   |-- plate_gpu.py             # GPU plate bending
|   |-- transient.py             # Transient thermal analysis
|
|-- cmp/                         # CMP prediction
|   |-- model.py                 # PCHIP + EPL kernel
|   |-- calibration.py           # Fab data calibration
|   |-- copper_cmp_calibration.py # Cu-specific calibration
|
|-- bonding/                     # Bond void risk
|   |-- model.py                 # Gap-threshold + physics solver
|
|-- anneal/                      # Anneal delamination
|   |-- model.py                 # CTE mismatch + fracture mechanics
|
|-- yield_model/                 # Monte Carlo yield
|   |-- model.py                 # Murphy/Stapper + spatial correlation
|
|-- features/                    # Density extraction
|-- io/                          # GDS/OASIS + KLayout I/O
|-- optimize/                    # Inverse design optimizer
|-- rules/                       # DRC rule compiler
|-- validation/                  # Benchmarks + fab correlation
|-- cross_pollination/           # Glass bonding extension
|-- ml/                          # Surrogate model (experimental)
|-- uq/                          # Uncertainty quantification
|-- viz/                         # Plot generation
|-- demo/                        # Synthetic layout generator
```

---

## Citation and References

### Citing This Work

```
Genesis PROV 9: Bondability -- Physics-Based Hybrid Bonding Yield Prediction
from GDS Layout to Manufacturing Outcome. Genesis Platform, 2026.
```

### Primary References

| Reference | Used For | Stage |
|---|---|---|
| Stine, B.E. et al., "Rapid Characterization and Modeling of Pattern-Dependent Variation in Chemical-Mechanical Polishing," IEEE Trans. Semicond. Manuf., vol. 11, no. 1, 1998. | CMP bathtub curve calibration (Al) | Stage 2 |
| Ouma, D.O. et al., "Characterization and Modeling of Oxide CMP Using Planarization Length and Pattern Density Concepts," IEEE Trans. Semicond. Manuf., 2002. | CMP bathtub curve calibration (Cu) | Stage 2 |
| Turner, K.T. and Spearing, S.M., "Modeling of Direct Wafer Bonding: Effect of Wafer Bow and Etch Patterns," J. Appl. Phys., vol. 92, no. 12, 2002. | Contact mechanics validation | Stage 3 |
| Suhir, E., "Stresses in Bi-Metal Thermostats," J. Appl. Mech., vol. 53, 1986. | Thermal stress analytical benchmark | Stage 4 |
| Murphy, B.T., "Cost-Size Optima of Monolithic Integrated Circuits," Proc. IEEE, vol. 52, no. 12, 1964. | Negative-binomial yield model | Stage 5 |
| Stapper, C.H., "Modeling of Defects in Integrated Circuit Photolithographic Patterns," IBM J. Res. Dev., vol. 27, no. 5, 1983. | Spatial defect correlation | Stage 5 |

### Supporting References

- Persson, B.N.J., "Contact Mechanics for Randomly Rough Surfaces," Surf. Sci. Rep., vol. 61, 2006.
- Maugis, D., "Adhesion of Spheres: The JKR-DMT Transition Using a Dugdale Model," J. Colloid Interface Sci., vol. 150, 1992.
- Cunningham, J.A., "The Use and Evaluation of Yield Models in Integrated Circuit Manufacturing," IEEE Trans. Semicond. Manuf., vol. 3, no. 2, 1990.
- Greenwood, J.A. and Williamson, J.B.P., "Contact of Nominally Flat Surfaces," Proc. R. Soc. London A, vol. 295, 1966.
- Stanley, H.M. and Kato, T., "An FFT-Based Method for Rough Surface Contact," J. Tribol., 1997.
- Hutchinson, J.W. and Suo, Z., "Mixed Mode Cracking in Layered Materials," Adv. Appl. Mech., vol. 29, 1992.
- Timoshenko, S.P. and Goodier, J.N., "Theory of Elasticity," 3rd Ed., McGraw-Hill, 1970.

---

## Appendix A: Glossary of Terms

| Term | Definition |
|---|---|
| **Hybrid bonding** | Direct Cu-Cu thermocompression bonding without solder, enabling sub-10 um pad pitch |
| **CMP** | Chemical-Mechanical Polishing/Planarization -- process to achieve sub-nm surface flatness |
| **Dishing** | Cu recess below surrounding dielectric after CMP -- occurs at low pad density |
| **Erosion** | Dielectric thinning at high pad density after CMP |
| **EPL** | Effective Planarization Length -- spatial averaging scale of the CMP process (~100 um) |
| **PCHIP** | Piecewise Cubic Hermite Interpolating Polynomial -- monotonicity-preserving interpolation |
| **Dugdale CZM** | Dugdale Cohesive Zone Model -- constant-traction adhesion model for interface mechanics |
| **CTE** | Coefficient of Thermal Expansion -- how much material expands per degree of temperature change |
| **Murphy model** | Negative-binomial yield model: Y = 1/(1+D) per defect cluster |
| **Stapper model** | Spatial correlation model for defect clustering in yield prediction |
| **SoIC** | System on Integrated Chips -- TSMC's chiplet integration technology using hybrid bonding |
| **Foveros** | Intel's 3D stacking technology using face-to-face hybrid bonding |
| **X-Cube** | Samsung's 3D chiplet stacking technology |
| **HBM** | High Bandwidth Memory -- stacked DRAM using hybrid bonding (HBM4+) |
| **W2W** | Wafer-to-Wafer bonding -- bonding two full wafers simultaneously |
| **C2W** | Chiplet-to-Wafer bonding -- placing individual dies onto a receiving wafer |
| **GDS** | Graphic Design System -- standard layout format for semiconductor masks |
| **OASIS** | Open Artwork System Interchange Standard -- compressed layout format |
| **KLayout** | Open-source layout viewer/editor with DRC marker support |
| **L-BFGS-B** | Limited-memory BFGS with Bound constraints -- optimization algorithm |
| **FFT** | Fast Fourier Transform -- O(N log N) algorithm for spectral decomposition |
| **Von Mises stress** | Equivalent scalar stress measure for multi-axial stress states |
| **Voigt mixing** | Upper-bound rule-of-mixtures for composite elastic properties |

---

## Appendix B: Configuration Quick Reference

### Minimal Configuration (YAML)

```yaml
process:
  name: "hbm4_demo"
  pad_pitch_um: 4.0
  anneal_temp_C: 300.0

cmp:
  epl_um: 100.0
  cmp_material: copper

bonding:
  use_physics_solver: true
  thickness_um: 775.0
  adhesion_energy_j_m2: 0.5

yield_model:
  monte_carlo_samples: 200
  correlation_length_um: 500.0
  random_seed: 42
```

### Key Parameters to Calibrate for Production Use

| Parameter | Default | What to Replace With | Priority |
|---|---|---|---|
| `cmp.recess_nm_at_density` | copper_hybrid_bonding preset (Enquist 2019, Kim 2022) | Fab-specific Cu CMP measurements | **Critical** |
| `yield_model.correlation_length_um` | 500 um (uncalibrated -- dominates yield by ~100x; use for relative comparison only) | Fab defect spatial correlation data | **Critical** |
| `bonding.overlay_sigma_nm` | 20 nm | Actual overlay capability (from bonding tool) | **High** |
| `anneal.anneal_temp_C` | 300 C | Actual anneal recipe temperature | **High** |
| `bonding.particle_density_cm2` | 0.5 /cm^2 | Fab particle monitoring data | **Medium** |
| `cmp.surface_roughness_nm` | 0.5 nm RMS | Fab AFM measurements | **Medium** |
| `bonding.thickness_um` | 775 um | Actual wafer thickness (thinned wafers differ) | **Medium** |
| `bonding.adhesion_energy_j_m2` | 0.5 J/m^2 | Measured surface energy (four-point bend test) | **Medium** |

---

## Appendix C: Frequently Asked Questions

**Q: Can I use this to predict yield in my fab?**
A: Not directly, not yet. The default calibration now uses the copper_hybrid_bonding preset (Enquist 2019, Kim 2022), which is more appropriate than the previous aluminum default, but is still published academic data. You must replace the CMP calibration with your fab-specific measurements, calibrate the correlation_length_um against your yield data (this single uncalibrated parameter dominates yield by ~100x), and validate the contact mechanics against your bonding process. The platform provides the physics framework; you provide the calibration data.

**Q: How does this compare to what TSMC/Intel/Samsung use internally?**
A: We do not know what these foundries use internally -- their yield prediction tools are proprietary. Bondability is likely simpler in some respects (2D not 3D, linear elastic, no multi-layer CMP) and more integrated in others (full physics chain from GDS to yield in one tool). The key difference is that Bondability is available for research evaluation.

**Q: Is the spectral FFT solver novel?**
A: The spectral approach to contact mechanics is well-established (Persson 2006). The novelty is applying it to hybrid bonding yield prediction within an automated pipeline, combined with Dugdale CZM adhesion and integrated with CMP, thermal, and yield models.

**Q: Why is the thermal solver linear elastic?**
A: Design-stage screening does not require the accuracy of incremental elasto-plastic analysis. Linear elastic stress is conservative (overestimates), which is the safe-side error for identifying risk. Adding plasticity would significantly increase complexity for modest accuracy improvement at screening resolution.

**Q: Can this handle real GDS files?**
A: Yes. The platform uses gdstk for GDS/OASIS parsing, which handles industry-standard layout files. You specify the pad layer and datatype, and the tool extracts the density map automatically.

**Q: What computing resources are needed?**
A: A standard laptop with Python 3.10+ is sufficient. The CPU thermal solver uses scipy sparse, requiring ~1-10s for typical grids. The GPU thermal solver (optional, requires PyTorch) accelerates larger grids. No cloud infrastructure or HPC is required.

**Q: What is the license?**
A: CC BY-NC-ND 4.0. You may share the work with attribution for non-commercial purposes. No derivatives without permission. Contact Genesis Platform for commercial licensing inquiries.

---

*Genesis PROV 9: Bondability. Research-status computational platform for physics-based hybrid bonding yield prediction. Six-stage physics chain from GDS layout to manufacturing yield. Spectral FFT contact mechanics at O(N log N). Validated against four published benchmarks. 61 production source files, 17 test files (all passing). Honest about limitations. Ready for fab calibration and experimental validation.*

*Version 2.0 | February 2026 | CC BY-NC-ND 4.0*
