# Bondability-D2W: Patent Data Room -- Comprehensive Technical Inventory

> Multi-physics simulation platform for hybrid bonding (Cu-Cu direct bonding)
> yield prediction in semiconductor manufacturing.

**Version:** 2.0.0 | **License:** MIT | **Python:** >=3.10

**Codebase:** ~61 production Python source files, ~13,000 LOC
**Tests:** 17 test files, all passing
**Interfaces:** 7 CLI commands, 5 REST API endpoints, Python API

---

## Table of Contents

1.  [Executive Summary](#1-executive-summary)
2.  [Quick Start](#2-quick-start)
3.  [Physics Pipeline](#3-physics-pipeline)
4.  [Complete Directory Tree](#4-complete-directory-tree)
5.  [Complete Python File Inventory](#5-complete-python-file-inventory)
6.  [Output Files Inventory](#6-output-files-inventory)
7.  [CLI Reference](#7-cli-reference)
8.  [REST API Reference](#8-rest-api-reference)
9.  [Python API Reference](#9-python-api-reference)
10. [Physics Solvers Detail](#10-physics-solvers-detail)
11. [Configuration Reference](#11-configuration-reference)
12. [Test Suite](#12-test-suite)
13. [Validation Benchmarks](#13-validation-benchmarks)
14. [Dependencies](#14-dependencies)
15. [Known Limitations](#15-known-limitations)
16. [Buyer Due Diligence Checklist](#16-buyer-due-diligence-checklist)

---

## 1. Executive Summary

### What Bondability-D2W Does

Bondability-D2W predicts the yield of hybrid-bonded (Cu-Cu direct bonding)
semiconductor dies by simulating the coupled physics chain from GDS layout
through manufacturing to final yield:

```
GDS/OASIS Layout
    |
    v
Feature Extraction (pad density, motifs, gradients)
    |
    v
CMP Recess Prediction (PCHIP + EPL kernel)
    |
    v
Contact Mechanics (Spectral FFT + Dugdale cohesive zone)
    |
    v
Thermal Stress Analysis (2D plane-stress FEA, CPU + GPU)
    |
    v
Monte Carlo Yield (Murphy/Stapper model with spatial correlation)
    |
    v
DRC Rule Compilation + Inverse Design Optimization
    |
    v
HTML Signoff Report + KLayout DRC Markers + Risk Maps
```

### Value Proposition

Hybrid bonding is the critical interconnect technology for advanced 3D
stacking (HBM, chiplets, CoWoS). Current industry practice relies on
post-silicon yield learning -- expensive and slow. Bondability-D2W moves
yield prediction to the design stage by:

1. **Physics-based prediction**, not empirical curve fitting. Each solver
   implements published, peer-reviewed models (Stine 1998, Turner 2002,
   Suhir 1986, Murphy 1964, Stapper 1983).

2. **GDS-in, yield-out pipeline**. Accepts real GDSII/OASIS layouts via
   gdstk, extracts pad density maps, and produces per-tile risk maps,
   yield distributions, and actionable DRC markers.

3. **Inverse design optimizer**. Multi-objective fill optimization that
   jointly minimizes void risk, delamination risk, CMP non-uniformity,
   and thermal stress. Both additive (hill-climbing) and subtractive
   (stress-minimizing) strategies.

4. **Self-validation**. Built-in benchmarks that compare solver outputs
   against published data with hold-out validation (not circular).

5. **Production-grade interfaces**. CLI (Typer + Rich), REST API
   (FastAPI + Swagger), Python API, and HTML report generation.

### Critical Calibration Notice

**CMP DEFAULT CALIBRATION: `copper_hybrid_bonding` preset.** The default
recess-vs-density curve now uses the `copper_hybrid_bonding` CMP preset
(recess ~2.5 nm at optimal density, from Enquist 2019, Kim 2022). The
original Stine et al. 1998 aluminum calibration is still available as a
fallback. **For production deployment, replace with fab-specific Cu CMP
measurements** using the built-in calibration module
(`bondability.cmp.calibration`) or manual YAML editing. See Section 15.1.

### What This Is NOT

This is not a first-principles CMP simulator (no Preston equation, no
slurry chemistry). It is not a TCAD tool. The CMP module is a calibrated
empirical model. The thermal solver is linear elastic (no plasticity).
The contact solver uses Dugdale approximation (not full JKR). These are
documented, understood, and appropriate trade-offs for the design-stage
screening use case.

---

## 2. Quick Start

### Installation

```bash
# From source (core dependencies only)
pip install -e .

# With development tools (pytest, ruff, mypy)
pip install -e ".[dev]"

# With GPU support (PyTorch for thermal solver)
pip install -e ".[opt]"

# With REST API server (FastAPI + uvicorn)
pip install -e ".[api]"

# Everything
pip install -e ".[all]"
```

### Run the Full Pipeline on a Density Array

```bash
bondability run --input examples/demo.npy --output results/
```

### Run on a GDS Layout

```bash
bondability run --input design.gds --layer 10 --config configs/process_example.yaml --output results/
```

### Run with Fill Optimization

```bash
bondability run --input layout.npy --config configs/process_example.yaml --optimize --output results/
```

### Run Validation Benchmarks

```bash
bondability benchmark
```

### Start the REST API Server

```bash
bondability serve --host 0.0.0.0 --port 8000
# Swagger docs at http://0.0.0.0:8000/docs
```

### Python API

```python
from bondability.pipeline import run_pipeline
from pathlib import Path

result = run_pipeline(
    gds_path=Path("design.gds"),
    process_yaml=Path("configs/process_example.yaml"),
    out_dir=Path("results/"),
    pad_layer=10,
    tile_um=25.0,
)

print(f"Yield P50: {result['yield_summary']['yield_p50']:.1%}")
print(f"Void hotspots: {result['rules_summary']['hot_void_frac']:.1%}")
```

---

## 3. Physics Pipeline

The pipeline runs six stages sequentially. Each stage consumes outputs
from prior stages and produces typed dictionaries for downstream stages.

### Stage 1: Feature Extraction

- **Input:** GDS/OASIS file (via gdstk) or pre-computed .npy density array.
- **Output:** `FeatureSet` dataclass with density map, density gradient,
  motif masks (isolated pads, TSV proximity, extreme gradient regions),
  pad count, bounding box.
- **Tile grid:** Configurable (default 25 um), defines spatial resolution
  for all downstream solvers.

### Stage 2: CMP Recess Prediction

- **Input:** FeatureSet, CMPConfig.
- **Physics:** Effective Planarization Length (EPL) kernel convolution of
  raw density, then PCHIP interpolation from density to recess (nm).
- **Output:** `eff_density`, `recess_mean_nm`, `recess_sigma_nm`, `gap_nm`,
  `cmp_margin_index`, and `explain` breakdown (density deficit, gradient
  magnitude, dominant driver mask).
- **Calibration:** Default 5-point bathtub curve from Stine et al. 1998.
  **MUST be replaced with fab-specific data for production use.**

### Stage 3: Bond Void Risk (Contact Mechanics)

- **Input:** FeatureSet, CMP output, BondingConfig.
- **Physics:** Spectral FFT contact solver with Dugdale cohesive zone
  (optional, controlled by `use_physics_solver`). Falls back to heuristic
  gap-threshold model if disabled.
- **Output:** `void_risk`, `open_bond_risk`, `gap_nm` (from solver),
  `explain` (overlay sigma map, risk decomposition).

### Stage 4: Anneal Delamination Risk (Thermal Stress)

- **Input:** FeatureSet, AnnealConfig.
- **Physics:** CTE mismatch stress (Cu: 17 ppm/K vs. SiO2: 0.5 ppm/K)
  with optional full thermal FEA. Voigt/Reuss mixing rules for composite
  modulus. Fracture mechanics interface flaw model.
- **Output:** `stress_index`, `delam_risk`.

### Stage 5: Monte Carlo Yield Distribution

- **Input:** FeatureSet, CMP output, bonding output, anneal output,
  YieldModelConfig.
- **Physics:** Murphy negative-binomial model (Stapper 1983) with spatial
  correlation via configurable correlation length. Monte Carlo sampling
  over overlay sigma, particle density, and surface roughness distributions.
  Edge exclusion zones. Per-cluster yield calculation.
- **Output:** `samples` (yield array), `summary` (P10/P50/P90/mean),
  `sensitivity` (elasticity analysis for 5 parameters).
- **CRITICAL TUNABLE:** `correlation_length_um` (default 500 um) is the
  single most impactful parameter on yield. Must be calibrated to fab data.

### Stage 6: DRC Rule Compilation

- **Input:** FeatureSet, CMP/bonding/anneal outputs, RulesConfig.
- **Output:** Violation masks (.npy), rules summary (JSON), suggestions
  list, KLayout marker database (.lyrdb).

### Stage 7: Report Generation

- **Output:** Self-contained HTML report with embedded base64 images,
  executive pass/fail verdict, yield metrics, sensitivity tornado chart,
  risk map gallery, DRC summary, and provenance timestamp.

---

## 4. Complete Directory Tree

```
PROV_9_BONDABILITY/
|
|-- README.md                        # This document
|-- LICENSE                          # MIT License
|-- CHANGELOG.md                     # Version history
|-- Makefile                         # Build/test/lint targets
|-- pyproject.toml                   # Package metadata, dependencies, tool config
|-- .pre-commit-config.yaml          # Pre-commit hooks (ruff)
|-- .gitignore
|
|-- bondability/                     # *** PRODUCTION SOURCE CODE ***
|   |-- __init__.py                  # Package root
|   |-- cli.py                       # Typer CLI (7 commands)
|   |-- config.py                    # Pydantic config schemas (7 sections)
|   |-- pipeline.py                  # End-to-end orchestration
|   |-- report.py                    # HTML report generator
|   |-- audit.py                     # Audit trigger logic
|   |
|   |-- api/                         # REST API (FastAPI)
|   |   |-- __init__.py
|   |   |-- server.py                # FastAPI app, 5 endpoint routes
|   |   |-- jobs.py                  # Async job store
|   |   |-- schemas.py               # Pydantic request/response models
|   |
|   |-- physics/                     # Physics solvers
|   |   |-- thermal.py               # CPU thermal FEA (direct sparse)
|   |   |-- gpu_thermal.py           # GPU thermal FEA (PyTorch CG)
|   |   |-- gpu_thermal_3d.py        # Multi-layer 3D thermal solver
|   |   |-- contact.py               # Spectral contact mechanics (Dugdale)
|   |   |-- plate_gpu.py             # GPU plate bending solver
|   |   |-- transient.py             # Transient thermal analysis
|   |   |-- thermal_stress_reconciliation.py  # CPU/GPU solver reconciliation
|   |   |-- errors.py                # Solver error types
|   |
|   |-- cmp/                         # CMP recess prediction
|   |   |-- __init__.py
|   |   |-- model.py                 # PCHIP + EPL kernel model
|   |   |-- calibration.py           # Calibration data management
|   |   |-- copper_cmp_calibration.py  # Cu-specific CMP calibration
|   |
|   |-- bonding/                     # Bond void risk model
|   |   |-- __init__.py
|   |   |-- model.py                 # Gap-threshold + physics solver path
|   |
|   |-- anneal/                      # Anneal delamination risk
|   |   |-- __init__.py
|   |   |-- model.py                 # CTE mismatch + fracture mechanics
|   |
|   |-- yield_model/                 # Monte Carlo yield engine
|   |   |-- __init__.py
|   |   |-- model.py                 # Murphy/Stapper with spatial correlation
|   |
|   |-- features/                    # Density & feature extraction
|   |   |-- __init__.py
|   |   |-- extract.py               # FeatureSet from GDS/npy
|   |   |-- density.py               # Density computation utilities
|   |
|   |-- io/                          # GDS/OASIS I/O
|   |   |-- __init__.py
|   |   |-- gds.py                   # gdstk-based GDS parser
|   |   |-- klayout.py               # KLayout marker database export
|   |
|   |-- optimize/                    # Inverse design optimizer
|   |   |-- __init__.py
|   |   |-- engine.py                # FillOptimizer + SubtractiveOptimizer
|   |
|   |-- rules/                       # DRC rule compiler
|   |   |-- __init__.py
|   |   |-- compiler.py              # Threshold-based violation detection
|   |
|   |-- validation/                  # Benchmarks & fab correlation
|   |   |-- __init__.py
|   |   |-- benchmarks.py            # 4 published-data benchmarks
|   |   |-- fab_correlation.py        # Fab correlation checks
|   |   |-- multi_node_validation.py  # Multi-node validation
|   |   |-- production_correlation.py # Production correlation checks
|   |   |-- scalability_benchmark.py  # Scalability benchmarks
|   |
|   |-- verification/                # Buyer verification
|   |   |-- __init__.py
|   |   |-- run_buyer_verification.py # Automated verification script
|   |
|   |-- ml/                          # Surrogate model (experimental)
|   |   |-- __init__.py
|   |   |-- surrogate.py             # ML surrogate model
|   |   |-- train.py                 # Training pipeline
|   |
|   |-- calibration/                  # Calibration tooling
|   |   |-- __init__.py
|   |   |-- kla_integration.py       # KLA tool integration
|   |   |-- synthetic_demo.py        # Synthetic calibration demo
|   |
|   |-- cross_pollination/           # Cross-domain extensions
|   |   |-- __init__.py
|   |   |-- glass_bonding.py         # Glass bonding model
|   |   |-- glass_bonding_validation.py  # Glass bonding validation
|   |
|   |-- uq/                          # Uncertainty quantification
|   |   |-- __init__.py
|   |   |-- montecarlo.py            # MC sampling utilities
|   |
|   |-- demo/                        # Synthetic layout generator
|   |   |-- __init__.py
|   |   |-- synthetic.py             # SyntheticLayout class
|   |
|   |-- viz/                         # Plotting utilities
|       |-- __init__.py
|       |-- plots.py                 # Quicklook plot generation
|
|-- tests/                           # *** 17 TEST FILES ***
|   |-- test_smoke.py                # Basic import/instantiation
|   |-- test_cmp.py                  # CMP model unit tests
|   |-- test_copper_cmp.py           # Copper CMP calibration tests
|   |-- test_physics_contact.py      # Contact solver tests
|   |-- test_bonding_week3.py        # Bond void risk tests
|   |-- test_anneal_week4.py         # Anneal delamination tests
|   |-- test_yield_week5.py          # Yield model tests
|   |-- test_rules_week6.py          # DRC rule compiler tests
|   |-- test_optimize_week7.py       # Optimizer tests
|   |-- test_gds_ingestion.py        # GDS I/O tests
|   |-- test_glass_bonding.py        # Glass bonding cross-pollination tests
|   |-- test_multi_node.py           # Multi-node validation tests
|   |-- test_scalability.py          # Scalability benchmark tests
|   |-- test_cli.py                  # CLI integration tests
|   |-- test_api.py                  # REST API endpoint tests
|   |-- test_benchmarks.py           # Validation benchmark tests
|   |-- test_stress_adversarial.py   # Adversarial stress tests
|
|-- configs/                         # *** CONFIGURATION FILES ***
|   |-- process_example.yaml         # Fully documented example config
|   |-- process_calibrated.yaml      # Calibrated config variant
|
|-- examples/                        # *** EXAMPLE DATA ***
|   |-- README.md                    # Example usage guide
|   |-- demo.npy                     # Sample density array
|
|-- docs/                            # Documentation directory
|
|-- archive/                         # *** HISTORICAL PROOF ARCHIVE ***
|   |-- runs/                        # Complete pipeline run outputs (6 runs)
|   |   |-- gds_audit/              # GDS-based audit run
|   |   |-- gds_w2_audit/           # Week 2 GDS audit run
|   |   |-- week3_proof/            # Week 3 bonding proof run
|   |   |-- week4_proof/            # Week 4 anneal proof run
|   |   |-- week6_proof/            # Week 6 rules proof run
|   |   |-- proof_pack/             # Final proof package run
|   |
|   |-- out_physics_final/           # Physics-validated final output
|   |-- output/                      # Historical experiment outputs
|   |   |-- billion_dollar/
|   |   |-- cli_result/
|   |   |-- discoveries/
|   |   |-- e2e_final/
|   |   |-- e2e_proof/
|   |   |-- gromacs_adhesion/
|   |   |-- hero_assets/
|   |   |-- solver_reconciliation/
|   |   |-- v9_verification/
|   |   |-- v10_verification/
|   |
|   |-- scripts/                     # 45 historical proof/audit scripts
|   |-- ml_datasets/                 # ML training data (200 samples)
|   |-- data/calibration/            # Calibration data files
|   |-- docs_hype/                   # Historical documentation
|   |-- *.md                         # Historical audit/valuation reports
|   |-- *.log                        # Verification logs
|   |-- *.json                       # Verification reports
```

---

## 5. Complete Python File Inventory

### Production Source (bondability/)

| File | Purpose | Key Functions/Classes |
|------|---------|----------------------|
| `__init__.py` | Package root | Version string |
| `cli.py` | Typer CLI with 7 commands | `run`, `optimize`, `analyze`, `benchmark`, `correlate`, `config show`, `config validate`, `serve` |
| `config.py` | Pydantic v2 configuration schemas | `AppConfig`, `CMPConfig`, `BondingConfig`, `AnnealConfig`, `YieldModelConfig`, `RulesConfig`, `OptimizeConfig`, `ProcessConfig` |
| `pipeline.py` | End-to-end orchestration | `run_pipeline()`, `load_config()`, `validate_pipeline_inputs()` |
| `report.py` | Self-contained HTML report | `generate_html_report()` |
| `audit.py` | Audit trigger logic | `check_audit_triggers()`, `save_audit_requests()` |
| `api/server.py` | FastAPI application | 5 endpoint handlers, background task runners |
| `api/jobs.py` | In-memory async job store | `JobStore` class |
| `api/schemas.py` | Pydantic request/response models | `SimulationRequest`, `OptimizationRequest`, `ConfigValidateRequest`, `JobStatus`, `HealthResponse` |
| `physics/thermal.py` | CPU thermal FEA (direct sparse solver) | `solve_thermal_stress()`, `validate_against_suhir()`, `MaterialProps`, `COPPER`, `SILICON_DIOXIDE`, `SILICON` |
| `physics/gpu_thermal.py` | GPU thermal FEA (PyTorch CG) | `GPUThermalSolver` class |
| `physics/gpu_thermal_3d.py` | Multi-layer 3D thermal solver | `MultiLayerThermalSolver` class |
| `physics/contact.py` | Spectral contact mechanics | `solve_contact_mechanics()` (FFT + Dugdale) |
| `physics/plate_gpu.py` | GPU plate bending solver | GPU-accelerated plate mechanics |
| `physics/transient.py` | Transient thermal analysis | Time-stepping thermal solver |
| `physics/errors.py` | Solver error hierarchy | `SolverError` exception class |
| `cmp/model.py` | CMP recess predictor | `predict_cmp_recess()`, `cmp_sanity_sweep()`, PCHIP interpolator |
| `cmp/calibration.py` | CMP calibration data | Calibration management utilities |
| `bonding/model.py` | Bond void risk model | `predict_bond_void_risk()` |
| `anneal/model.py` | Anneal delamination model | `predict_anneal_delam_risk()` |
| `yield_model/model.py` | Monte Carlo yield engine | `predict_yield_distribution()` |
| `features/extract.py` | Feature extraction | `extract_features()`, `FeatureSet` dataclass |
| `features/density.py` | Density computation | Density map utilities |
| `io/gds.py` | GDS/OASIS parser | gdstk-based layout ingestion |
| `io/klayout.py` | KLayout export | `write_rdb()` for .lyrdb marker database |
| `optimize/engine.py` | Inverse design optimizer | `FillOptimizer`, `SubtractiveOptimizer` |
| `rules/compiler.py` | DRC rule compiler | `compile_rules()` |
| `validation/benchmarks.py` | Published-data benchmarks | `run_all_benchmarks()`, 4 benchmark functions |
| `validation/fab_correlation.py` | Fab correlation checks | `run_all_correlations()` |
| `verification/run_buyer_verification.py` | Automated buyer verification | End-to-end verification script |
| `ml/surrogate.py` | ML surrogate model | Surrogate model for fast inference |
| `ml/train.py` | ML training pipeline | Training loop and data loading |
| `uq/montecarlo.py` | MC sampling utilities | Uncertainty quantification helpers |
| `demo/synthetic.py` | Synthetic layout generator | `SyntheticLayout` class |
| `viz/plots.py` | Quicklook plot generation | `plot_quicklook()` |

### Test Files (tests/)

| File | Coverage Area | Key Assertions |
|------|---------------|----------------|
| `test_smoke.py` | Basic imports, config instantiation | Modules load, defaults valid |
| `test_cmp.py` | CMP model correctness | Bathtub shape, PCHIP monotonicity, recess bounds |
| `test_physics_contact.py` | Spectral contact solver | Convergence, bridging/conforming ordering |
| `test_bonding_week3.py` | Bond void risk pipeline | Risk bounds [0,1], motif penalty |
| `test_anneal_week4.py` | Anneal delamination pipeline | Stress index, CTE mismatch |
| `test_yield_week5.py` | Monte Carlo yield engine | Yield in [0,1], sensitivity monotonicity |
| `test_rules_week6.py` | DRC rule compiler | Violation masks, threshold behavior |
| `test_optimize_week7.py` | Fill optimizer | Yield improvement, fill budget |
| `test_gds_ingestion.py` | GDS I/O via gdstk | Layout parsing, density extraction |
| `test_cli.py` | CLI integration | Command execution, output file creation |
| `test_api.py` | REST API endpoints | HTTP status codes, response schemas |
| `test_benchmarks.py` | Validation benchmarks | All 4 benchmarks pass |
| `test_copper_cmp.py` | Copper CMP calibration | Cu recess curves, Preston equation, calibration pipeline |
| `test_glass_bonding.py` | Glass bonding cross-pollination | AGC/Corning/Schott properties, bond energy, yield prediction |
| `test_multi_node.py` | Multi-node validation | Distributed pipeline, cross-node consistency checks |
| `test_scalability.py` | Scalability benchmarks | Large-grid performance, memory usage, wall-time |
| `test_stress_adversarial.py` | Adversarial stress tests | Edge cases, extreme densities, NaN handling |

### Archive Scripts (archive/scripts/) -- 45 Historical Proof Scripts

These scripts are NOT part of the production pipeline. They are historical
proof-of-concept, audit, and validation scripts retained for provenance.

| File | Purpose |
|------|---------|
| `_audit_checker.py` | Internal audit validation |
| `active_learning_loop.py` | Active learning exploration |
| `attack_optimizer.py` | Adversarial optimizer testing |
| `check_dataset_stats.py` | ML dataset statistics |
| `debug_physics_v4.py` | Physics debugging (v4) |
| `deep_audit_all.py` | Comprehensive audit runner |
| `deep_proof.py` | Deep validation proof |
| `deep_sensitivity.py` | Sensitivity analysis |
| `demo_surrogate.py` | Surrogate model demo |
| `discover_design_rules.py` | Design rule discovery |
| `discovery_engine.py` | Automated discovery |
| `discovery_sweeps.py` | Parameter sweep discovery |
| `discovery_sweeps_345.py` | Extended parameter sweeps |
| `e2e_platform_proof.py` | End-to-end platform proof |
| `fidelity_audit.py` | Solver fidelity audit |
| `generate_billion_dollar_proof.py` | Value proof generation |
| `generate_hero_assets.py` | Showcase asset generation |
| `generate_ml_dataset.py` | ML training data generation |
| `generate_patent_fill.py` | Patent figure generation |
| `grid_convergence_proof.py` | Mesh convergence study |
| `gromacs_cu_adhesion.py` | GROMACS Cu adhesion simulation |
| `hyper_optimize.py` | Hyperparameter optimization |
| `mega_proof.py` | Comprehensive proof runner |
| `parameter_sweep_proof.py` | Parameter sweep proof |
| `physics_audit_final.py` | Final physics audit |
| `reconcile_solvers.py` | CPU/GPU solver reconciliation |
| `reliability_compiler.py` | Reliability analysis |
| `scaling_audit.py` | Performance scaling audit |
| `simulate_floorplan.py` | Floorplan simulation |
| `simulate_moats.py` | Moat structure simulation |
| `solver_consensus_proof.py` | Solver agreement proof |
| `stress_test_ip.py` | IP stress testing |
| `ultimate_coupling.py` | Multi-physics coupling test |
| `verify_3d_stack.py` | 3D stack verification |
| `verify_breakpoint.py` | Breakpoint verification |
| `verify_claims.py` | Claims verification (v1) |
| `verify_claims_v2.py` | Claims verification (v2) |
| `verify_full_reticle.py` | Full-reticle verification |
| `verify_gpu_pipeline.py` | GPU pipeline verification |
| `verify_grid_convergence.py` | Grid convergence verification |
| `verify_post_audit.py` | Post-audit verification |
| `verify_robustness_monte_carlo.py` | Monte Carlo robustness |
| `verify_v10_proof.py` | Version 10 proof |
| `verify_v11_physics.py` | Version 11 physics proof |
| `visualize_physics_deep_dive.py` | Physics visualization |

---

## 6. Output Files Inventory

### Pipeline Run Outputs

Each pipeline run (`bondability run`) produces the following directory
structure in the output directory:

```
results/
|-- report.html                       # Self-contained HTML signoff report
|-- yield_summary.json                # Yield P10/P50/P90/mean, MC params
|-- features_report.json              # Feature extraction summary
|-- rules_summary.json                # DRC violation summary
|-- suggestions.json                  # Actionable design suggestions
|-- audit_request.json                # Audit trigger flags
|-- optimization_report.json          # (if --optimize) Optimizer report
|-- pipeline_timing.json              # Per-stage timing breakdown
|-- markers.lyrdb                     # KLayout marker database (DRC)
|
|-- risk_maps/                        # Numpy arrays (.npy)
|   |-- density.npy                   # Raw pad density [H, W]
|   |-- density_grad.npy              # Density gradient magnitude
|   |-- eff_density.npy               # CMP effective density (EPL-smoothed)
|   |-- recess_mean_nm.npy            # CMP recess prediction (nm)
|   |-- recess_sigma_nm.npy           # CMP recess variation (nm)
|   |-- gap_nm.npy                    # Contact gap (nm)
|   |-- cmp_margin_index.npy          # CMP margin index [0=healthy, 1=critical]
|   |-- void_risk.npy                 # Bond void risk [0, 1]
|   |-- open_bond_risk.npy            # Overlay open-bond risk [0, 1]
|   |-- stress_index.npy              # Thermal stress index
|   |-- delam_risk.npy                # Delamination risk [0, 1]
|   |-- motif_*.npy                   # Motif detection masks
|   |-- cmp_explain/                  # CMP explainability
|       |-- density_deficit.npy       # Distance from 50% optimum density
|       |-- density_driven_mask.npy   # Dominant risk factor mask
|       |-- density_gradient_mag.npy  # Local gradient magnitude
|
|-- violations/                       # DRC violation masks (.npy)
|   |-- low_density.npy               # Below min density threshold
|   |-- high_density.npy              # Above max density threshold
|   |-- density_gradient.npy          # Exceeds max gradient
|   |-- void_hotspots.npy             # Void risk above threshold
|   |-- delam_hotspots.npy            # Delamination risk above threshold
|   |-- margin_critical.npy           # CMP margin critical zones
|
|-- plots/                            # PNG visualizations
    |-- density.png                   # Pad density heatmap
    |-- eff_density.png               # CMP effective density heatmap
    |-- recess_mean_nm.png            # CMP recess heatmap
    |-- void_risk.png                 # Void risk heatmap
    |-- delam_risk.png                # Delamination risk heatmap
    |-- open_bond_risk.png            # Open bond risk heatmap
    |-- stress_index.png              # Thermal stress heatmap
    |-- yield_histogram.png           # Monte Carlo yield distribution
    |-- sensitivity_tornado.png       # Parameter sensitivity tornado
    |-- cmp_margin.png                # CMP margin index heatmap
    |-- density_gradient.png          # Density gradient heatmap
```

### Archived Run Outputs (archive/runs/)

Six complete pipeline runs are preserved for provenance:

| Run Directory | Content | Status |
|--------------|---------|--------|
| `gds_audit/` | GDS-ingested audit run | Complete (risk maps, plots, violations, JSON) |
| `gds_w2_audit/` | Week 2 GDS audit run | Complete (with CMP explain maps) |
| `week3_proof/` | Week 3 bonding proof | Complete (contact mechanics validated) |
| `week4_proof/` | Week 4 anneal proof | Complete (thermal stress validated) |
| `week6_proof/` | Week 6 rules proof | Complete (DRC compilation validated) |
| `proof_pack/` | Final integrated proof | Complete (all stages, optimization report) |

Each archived run contains the full output structure: risk_maps/, plots/,
violations/, JSON summaries, and (where applicable) HTML reports and
KLayout markers.

### Archived Physics-Final Output (archive/out_physics_final/)

Complete physics-validated output with optimization report, including
all risk maps, CMP explainability arrays, violation masks, plots, and
HTML signoff report.

---

## 7. CLI Reference

The CLI is built with Typer and Rich. Entry point: `bondability`.

### `bondability run`

Run the full bondability pipeline on a layout.

```
Options:
  -i, --input PATH        Input .npy density or .gds/.oas layout file [required]
  -c, --config PATH       Process config YAML (optional, uses defaults)
  -o, --output PATH       Output directory [default: results]
  -l, --layer INT         GDS pad layer (required for .gds input)
  --datatype INT          GDS pad datatype [default: 0]
  --tile-um FLOAT         Tile size in microns [default: 25.0]
  --optimize              Enable fill optimization
  -v, --verbose           Verbose logging
```

Stages executed: Feature extraction -> CMP -> Bonding -> Anneal -> Yield -> Rules -> Save + Report.

### `bondability optimize`

Run fill optimization on a density map (standalone, not part of full pipeline).

```
Options:
  -i, --input PATH        Input .npy density file [required]
  -o, --output PATH       Output directory [default: result]
  -m, --method TEXT       "additive" or "subtractive" [default: subtractive]
  -n, --iterations INT    Max iterations [default: 30]
  --restarts INT          Multi-start restarts [default: 5]
  --no-gpu                Disable GPU thermal in optimization loop
  --prefill / --no-prefill  Apply checker prefill [default: True]
  -v, --verbose           Verbose logging
```

Outputs: `optimized_layout.npy`, `report.json` with yield, void risk,
delam risk, GPU peak stress, and elapsed time.

### `bondability analyze`

Thermal stress analysis (2D single-layer or 3D multi-layer).

```
Options:
  -i, --input PATH        Input .npy density file [required]
  --layers INT            Number of layers [default: 1]
  --layer-files TEXT      Per-layer .npy file paths (optional)
  --compare-smooth        Compare sharp vs. smoothed interface
  -v, --verbose           Verbose logging
```

For `--layers 1`: uses 2D GPU thermal solver.
For `--layers > 1`: uses MultiLayerThermalSolver with via stress analysis.

### `bondability benchmark`

Run self-validation benchmarks against published data. Exits with code 1
if any benchmark fails.

```
Options:
  -v, --verbose           Verbose logging
```

Runs 4 benchmarks: CMP Stine 1998 (hold-out), Contact Turner 2002,
Thermal Suhir 1986, Yield sanity + seed configurability.

### `bondability correlate`

Run fab correlation checks against published literature.

```
Options:
  -o, --output PATH       Output directory [default: output]
  -v, --verbose           Verbose logging
```

Outputs: `fab_correlation_report.json`.

### `bondability config show`

Print the default configuration as JSON to stdout.

### `bondability config validate <path>`

Validate a process configuration YAML file against the Pydantic schema.
Reports process name, CMP EPL, anneal temperature, and MC sample count.

### `bondability serve`

Start the REST API server (requires `bondability-d2w[api]`).

```
Options:
  --host TEXT             Bind address [default: 127.0.0.1]
  -p, --port INT          Port [default: 8000]
  -w, --workers INT       Uvicorn workers [default: 1]
  --reload                Auto-reload on code changes
```

---

## 8. REST API Reference

Start with `bondability serve`. Interactive Swagger docs at `/docs`.

### POST /api/v1/simulate

Submit a simulation job (asynchronous).

**Request body:**
```json
{
  "density": [[0.3, 0.5, ...], [0.4, 0.6, ...]],
  "tile_um": 25.0,
  "config_overrides": {"cmp": {"epl_um": 80}}
}
```

**Response:** `JobStatus` with `job_id` and `state: "pending"`.

**Background:** Runs full pipeline (CMP -> bonding -> anneal -> yield -> rules).

### GET /api/v1/simulate/{job_id}

Poll job status. Returns `JobStatus` with `state` ("pending", "running",
"completed", "failed"), `progress` string, and `result` dict on completion.

**Result payload on completion:**
```json
{
  "yield_summary": {"yield_p10": 0.85, "yield_p50": 0.91, ...},
  "rules_summary": {"hot_void_frac": 0.02, ...},
  "suggestions": [...],
  "grid_size": [64, 64]
}
```

### POST /api/v1/optimize

Submit an optimization job (asynchronous).

**Request body:**
```json
{
  "density": [[...]],
  "tile_um": 25.0,
  "method": "subtractive",
  "iterations": 15,
  "config_overrides": {}
}
```

### GET /api/v1/optimize/{job_id}

Poll optimization job status. Result includes `yield_summary`,
`optimized_density_mean`, `void_risk_mean`, `delam_risk_mean`, `method`.

### POST /api/v1/validate

Validate a configuration dictionary against the Pydantic schema.

**Request body:**
```json
{
  "config": {"cmp": {"epl_um": 80}, "anneal": {"anneal_temp_C": 350}}
}
```

**Response:**
```json
{"valid": true, "effective_config": {...}}
```

### GET /api/v1/health

Health check. Returns solver availability (GPU thermal, GDS parser) and
GPU device type.

### GET /api/v1/benchmarks

Run all validation benchmarks and return the full report JSON.

---

## 9. Python API Reference

### Primary Entry Point: `run_pipeline()`

```python
from bondability.pipeline import run_pipeline
from pathlib import Path

result = run_pipeline(
    gds_path=Path("design.gds"),      # GDS/OAS file (or None for synthetic)
    process_yaml=Path("config.yaml"),  # Process configuration
    out_dir=Path("results/"),          # Output directory
    pad_layer=10,                      # GDS layer for pads
    pad_datatype=0,                    # GDS datatype for pads
    tile_um=25.0,                      # Tile resolution (microns)
    layout_override=None,              # Optional SyntheticLayout
    optimize=False,                    # Enable fill optimization
    dry_run=False,                     # Validate without solving
)

# result keys: "features", "yield_summary", "rules_summary", "timings"
# If errors occurred: result["errors"] lists them
```

### Individual Solver APIs

```python
# CMP
from bondability.cmp.model import predict_cmp_recess, cmp_sanity_sweep
cmp_out = predict_cmp_recess(features, cmp_config)

# Contact mechanics
from bondability.physics.contact import solve_contact_mechanics
result = solve_contact_mechanics(
    topology_nm, pixel_size_um=25.0, modulus_gpa=130.0,
    thickness_um=775.0, adhesion_energy_j_m2=0.5,
    adhesion_range_nm=5.0, refine_factor=4,
)

# Thermal stress (CPU)
from bondability.physics.thermal import solve_thermal_stress
result = solve_thermal_stress(
    density, tile_um=25.0, delta_T_K=275.0,
    boundary="free", smooth_interface=False,
)

# Thermal stress (GPU)
from bondability.physics.gpu_thermal import GPUThermalSolver
solver = GPUThermalSolver()
result = solver.solve(density, tile_um=25.0, delta_T_K=275.0)

# Yield
from bondability.yield_model.model import predict_yield_distribution
yield_out = predict_yield_distribution(features, cmp_out, bonding_out, anneal_out, yield_config)

# Optimizer
from bondability.optimize.engine import FillOptimizer, SubtractiveOptimizer
optimizer = FillOptimizer(app_config)
opt_features, report = optimizer.optimize(features)
```

### Configuration API

```python
from bondability.config import AppConfig
from bondability.pipeline import load_config

# Load from YAML
cfg = load_config(Path("configs/process_example.yaml"))

# Create with defaults
cfg = AppConfig()

# Access sections
cfg.cmp.epl_um          # 100.0
cfg.bonding.gap_nm_threshold  # 12.0
cfg.yield_model.correlation_length_um  # 500.0

# Serialize
json_str = cfg.model_dump_json(indent=2)
```

---

## 10. Physics Solvers Detail

### 10.1 CMP Solver: PCHIP Interpolation with EPL Kernel

**File:** `bondability/cmp/model.py`

**Algorithm:**
1. Compute effective density by convolving raw pad density with a Gaussian
   kernel whose sigma is derived from the Effective Planarization Length
   (EPL, default 100 um). The EPL represents the pad-scale averaging
   behavior of CMP. Supports optional anisotropic kernels.
2. Map effective density to recess (nm) via PCHIP (Piecewise Cubic Hermite
   Interpolating Polynomial) interpolation. PCHIP preserves monotonicity
   and avoids overshoot -- critical for physical sanity.
3. Compute density-dependent recess sigma:
   `sigma(d) = base_sigma * (1 + scale * (1 - d))`.
   Higher variation at low density (isolated features).
4. Derive gap proxy: `gap_nm = recess_mean + surface_roughness`.
5. Compute CMP margin index: `gap / gap_threshold`, clipped to [0, 1].

**Calibration Data:**
Default 5-point bathtub curve from Stine et al. 1998:
- d=0.0 -> 25 nm (high dishing at very low density)
- d=0.3 -> 10 nm
- d=0.5 -> 5 nm (minimum recess near optimal density)
- d=0.7 -> 8 nm (erosion begins)
- d=1.0 -> 20 nm (significant erosion at very high density)

**Note:** The default is now the `copper_hybrid_bonding` preset (Enquist 2019,
Kim 2022). The original Stine 1998 aluminum curve is available as fallback.
For production use, replace with fab-specific measurements via the calibration
module or manual YAML editing.

**Reference:** Stine et al., "Rapid Characterization and Modeling of
Pattern-Dependent Variation in CMP", IEEE TSM, 1998.

### 10.2 Contact Solver: Spectral FFT with Dugdale Cohesive Zone

**File:** `bondability/physics/contact.py`

**Algorithm:**
1. Optional mesh refinement: subdivide tiles into sub-um elements
   (bilinear interpolation, configurable refine_factor).
2. Nondimensionalize all quantities (characteristic pressure = sigma_max,
   characteristic length = dc).
3. Build spectral kernel combining plate bending stiffness (D * K^4) and
   surface compliance (E*/(2*dx)), in serial compliance.
4. Formulate energy functional: elastic energy (spectral) + Dugdale
   adhesion energy (smoothed piecewise-linear with Huber-like softplus
   approximation, alpha=20 for near-step behavior).
5. Solve with L-BFGS-B (box constraints: displacement in [0, gap] for
   peaks, zero for valleys).
6. Convert back to physical units (nm). Block-average to tile resolution
   if mesh was refined.

**Physics basis:**
- Kirchhoff plate bending: `D = E*h^3 / (12*(1-nu^2))` for flexural
  rigidity of the wafer.
- Dugdale cohesive zone: constant traction `sigma_max = gamma/dc` for
  gap < dc, zero for gap > dc. Standard Maugis-Dugdale model.
- Spectral method: O(N log N) via FFT. The kernel in Fourier space
  decouples the 2D problem into independent modes.

**Complexity:** O(N log N) per L-BFGS-B iteration, where N = H*W (or
refined H*W*rf^2). Typical convergence in 50-200 iterations.

**HONEST LIMITATION:** At tile resolution (25 um), the Dugdale cohesive
zone (dc=5 nm) is severely under-resolved. The solver warns about this.
Use refine_factor >= 4 for production accuracy (16x slower). Qualitative
ordering (thick bridges, thin conforms) is correct at any resolution.

**Reference:** Turner & Spearing, "Modeling of Direct Wafer Bonding",
J. Appl. Phys., 2002.

### 10.3 Thermal Solver: 2D Plane-Stress FEA

**Files:** `bondability/physics/thermal.py` (CPU), `bondability/physics/gpu_thermal.py` (GPU)

**CPU Variant (Direct Sparse Solver):**
1. Assign material properties per tile via Voigt rule-of-mixtures:
   density=1 -> Cu, density=0 -> SiO2. Computes E, nu, CTE fields.
2. Compute plane-stress stiffness: C11 = E/(1-nu^2), C12 = nu*E/(1-nu^2),
   C66 = E/(2*(1+nu)).
3. Thermal strain: eps_th = CTE * delta_T.
4. Assemble sparse linear system (2 DOFs per node: u, v) using central
   finite differences for equilibrium equations.
5. Direct solve via `scipy.sparse.linalg.spsolve` -- guaranteed
   convergence, no iteration count worries.
6. Extract displacement fields, compute strain and stress from displacements,
   compute von Mises equivalent stress.
7. Optional smooth_interface: 1-tile Gaussian smoothing of density at
   material interfaces eliminates the sharp-interface FD stress singularity.

**GPU Variant (PyTorch Conjugate Gradient):**
1. Same physics as CPU but implemented in PyTorch tensors.
2. Conjugate gradient solver with configurable max_iter and tolerance.
3. Supports MPS (Apple Silicon), CUDA, and CPU fallback.
4. Boundary options: "free" (natural BCs) or "clamped" (zero displacement).

**3D Multi-Layer Variant:** `gpu_thermal_3d.py` extends to multiple bonded
layers with inter-layer via stress analysis.

**Key difference between CPU and GPU:** CPU defaults to free BCs, GPU
defaults to free BCs (configurable). Clamped BCs model a die constrained
by surrounding wafer material. Free BCs model an isolated die. Different
BCs produce different stress distributions; ensure matching when comparing.

**HONEST LIMITATION:** Linear elastic only. No plasticity. At 300C anneal,
peak von Mises stress at sharp Cu/SiO2 interfaces can exceed Cu yield
(~300 MPa). In reality, Cu yields and stress redistributes. Implementing
proper incremental plasticity would require load stepping, tangent stiffness
updates, and deviatoric radial return. The solver reports honest elastic
stress and documents this limitation.

**Reference:** Suhir, "Stresses in Bi-Metal Thermostats", J. Appl. Mech.,
1986.

### 10.4 Yield Engine: Murphy/Stapper with Monte Carlo

**File:** `bondability/yield_model/model.py`

**Algorithm:**
1. Sample global process parameters (N Monte Carlo trials):
   - Overlay sigma: Normal(mu=20, sigma=3) nm, clipped to >= 5
   - Particle density: Normal(mu=0.5, sigma=0.1) defects/cm2
   - Surface roughness: Normal(mu=0.5, sigma=0.1) nm RMS
2. For each trial:
   a. Scale open-bond risk by (sampled_sigma / reference_sigma)^2.
   b. Perturb gap map with Gaussian noise (sigma = sampled roughness).
   c. Compute void risk via logistic around snap-in threshold (~9 nm).
   d. Combine per-tile failure: P_fail = 1 - (1-P_open)(1-P_void)(1-P_delam).
   e. Apply edge exclusion (default 2 tiles per edge).
   f. Partition interior into spatial clusters (n = H*W / corr_tiles^2).
   g. Per-cluster Murphy yield: Y_k = 1/(1+D_k) where D_k = mean risk.
   h. Die yield = product of cluster yields * Poisson particle yield.
3. Compute summary statistics: P10, P50, P90, mean yield.
4. Sensitivity analysis: quartile-based elasticity for overlay, particles,
   roughness, plus structural parameters (correlation length, edge exclusion).

**CRITICAL TUNABLE:** `correlation_length_um` (default 500 um). This
controls the number of independent failure clusters:
`n_clusters = (H*W) / (corr_tiles^2)`. Lower correlation -> more clusters
-> lower yield. Higher correlation -> fewer clusters -> higher yield.
This is the single most impactful parameter on final yield numbers.
Must be calibrated to fab-specific defect correlation data.

**References:**
- Murphy, "Yield, Reliability, and Defect Density", Proc. IEEE, 1964.
- Stapper, "Modeling of Defects in ICs", IBM J. Res. Dev., 1983.

### 10.5 Optimizer: Gradient-Free Inverse Design

**File:** `bondability/optimize/engine.py`

**Two optimizer strategies:**

**FillOptimizer (Additive Hill Climbing):**
1. Evaluate baseline multi-objective fitness:
   fitness = -(w_void*void + w_delam*delam + w_cmp*sigma/10 + w_thermal*stress/1000)
2. For each restart x each iteration:
   a. Identify highest combined-risk tile.
   b. Place Gaussian fill blob at hotspot (configurable sigma, amplitude).
   c. Check fill budget constraint.
   d. Accept if fitness improves.
3. Convergence patience: stop after N stalled iterations.
4. Multi-start: avoids local minima.

**SubtractiveOptimizer (Stress Minimizing):**
1. Start from checker fill base (50 um blocks, 40% fill strength).
2. Run GPU thermal solver to find stress hotspots.
3. Apply Gaussian carve-out at peak stress location.
4. Accept only if yield maintained within tolerance AND stress improved.
5. Adaptive: reduce carve amplitude on rejection.

**Multi-objective weights (all configurable):**
- `weight_void`: Bond void risk (default 1.0)
- `weight_delam`: Delamination risk (default 1.0)
- `weight_cmp_sigma`: CMP non-uniformity (default 0.5)
- `weight_thermal`: GPU thermal peak stress (default 0.3)

---

## 11. Configuration Reference

Configuration is managed via Pydantic v2 models in `bondability/config.py`.
All parameters have documented defaults. Configuration can be loaded from
YAML files or instantiated programmatically.

### Section 1: `process`

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `name` | str | "process" | Process identifier |
| `pad_pitch_um` | float | 4.0 | Pad pitch in microns |
| `anneal_temp_C` | float | 300.0 | Post-bond anneal temperature |
| `anneal_time_min` | float | 60.0 | Anneal duration (minutes) |

### Section 2: `cmp`

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `epl_um` | float | 100.0 | Effective Planarization Length (um) |
| `recess_nm_at_density` | Dict[float, float] | 5-point bathtub | Calibration: density -> recess (nm) |
| `recess_sigma_nm` | float | 3.0 | Base recess variation (nm) |
| `density_sigma_scale` | float | 0.5 | Density-dependent sigma scaling |
| `surface_roughness_nm` | float | 0.5 | RMS surface roughness (nm) |
| `gap_threshold_nm` | float | 15.0 | Gap threshold for CMP margin |

### Section 3: `bonding`

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `gap_nm_threshold` | float | 12.0 | Gap threshold for void risk |
| `overlay_sigma_nm` | float | 20.0 | Overlay sigma (nm) |
| `particle_ppm` | float | 10.0 | Particle contamination (ppm) |
| `particle_density_cm2` | float | 0.5 | Particle density (defects/cm2) |
| `particle_kill_radius_um` | float | 10.0 | Particle kill zone radius |
| `warpage_um` | float | 1.0 | Wafer bow (um) |
| `motif_penalty_factor` | float | 2.0 | Risk multiplier for bad motifs |
| `max_acceptable_bow_um` | float | 5.0 | SEMI M1-0302 bow limit |
| `use_physics_solver` | bool | True | Enable spectral contact solver |
| `modulus_gpa` | float | 130.0 | Young's modulus (Si) |
| `thickness_um` | float | 775.0 | Wafer thickness |
| `adhesion_energy_j_m2` | float | 0.5 | Surface energy (J/m2) |
| `poisson_ratio` | float | 0.28 | Si Poisson ratio |
| `adhesion_range_nm` | float | 5.0 | Dugdale cohesive zone width |
| `logistic_steepness` | float | 5.0 | Physics gap->risk steepness |
| `logistic_steepness_heuristic` | float | 0.3 | Heuristic path steepness |
| `gap_contact_threshold_nm` | float | 1.0 | Contact threshold |
| `pad_pitch_um` | float | 4.0 | Pad pitch for overlay model |
| `overlay_margin_ratio` | float | 0.4 | Fraction of pitch |
| `overlay_edge_multiplier` | float | 1.3 | Edge sigma increase |
| `wafer_diameter_mm` | float | 300.0 | Wafer diameter |

### Section 4: `anneal`

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `density_gradient_weight` | float | 1.0 | Gradient stress weight |
| `max_gradient_ok` | float | 0.08 | Acceptable gradient threshold |
| `stress_diffusion_length_um` | float | 10.0 | Stress diffusion length |
| `edge_stress_weight` | float | 0.5 | Edge stress contribution |
| `edge_decay_um` | float | 20.0 | Edge effect decay distance |
| `cte_cu_ppm_K` | float | 17.0 | Copper CTE (ppm/K) |
| `cte_dielectric_ppm_K` | float | 0.5 | SiO2 CTE (ppm/K) |
| `anneal_temp_C` | float | 300.0 | Anneal temperature (C) |
| `use_thermal_fea` | bool | False | Enable full thermal FEA |
| `interface_flaw_um` | float | 5.0 | Worst-case flaw size (um) |
| `mixing_rule` | str | "voigt" | "voigt" or "reuss" |

### Section 5: `yield_model`

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `monte_carlo_samples` | int | 200 | MC trial count |
| `sensitivity_analysis` | bool | True | Enable sensitivity elasticity |
| `random_seed` | int or None | 42 | Reproducibility seed |
| `dist_overlay_sigma` | (float, float) | (20.0, 3.0) | Overlay sigma distribution (mean, std) |
| `dist_particle_density` | (float, float) | (0.5, 0.1) | Particle density distribution |
| `dist_roughness_nm` | (float, float) | (0.5, 0.1) | Surface roughness distribution |
| `correlation_length_um` | float | 500.0 | **CRITICAL:** Spatial correlation length |
| `adhesion_range_nm` | float | 5.0 | For gap threshold derivation |
| `gap_threshold_nm` | float | 15.0 | Roughness perturbation threshold |
| `edge_exclusion_tiles` | int | 2 | Tiles to exclude per edge |

### Section 6: `rules`

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `tile_um` | float | 25.0 | Tile size (um) |
| `max_density_gradient` | float | 0.10 | Max gradient threshold |
| `min_density` | float | 0.15 | Minimum density rule |
| `max_density` | float | 0.85 | Maximum density rule |
| `void_risk_threshold` | float | 0.7 | Void hotspot threshold |
| `delam_risk_threshold` | float | 0.7 | Delamination hotspot threshold |
| `cmp_margin_threshold` | float | 0.8 | CMP margin critical threshold |

### Section 7: `optimize`

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `fill_sigma_tiles` | float | 2.0 | Gaussian blob sigma |
| `fill_amplitude` | float | 0.1 | Max density increment per blob |
| `max_fill_fraction` | float | 0.5 | Total fill budget |
| `convergence_tol` | float | 1e-4 | Stop threshold |
| `convergence_patience` | int | 3 | Stalled iteration patience |
| `iterations` | int | 15 | Max iterations per restart |
| `restarts` | int | 2 | Multi-start restarts |
| `weight_void` | float | 1.0 | Void risk objective weight |
| `weight_delam` | float | 1.0 | Delamination objective weight |
| `weight_cmp_sigma` | float | 0.5 | CMP uniformity weight |
| `weight_thermal` | float | 0.3 | Thermal stress weight |
| `use_gpu_thermal` | bool | True | GPU thermal in optimizer loop |
| `thermal_eval_interval` | int | 5 | Thermal eval every N steps |

---

## 12. Test Suite

### Test Inventory (17 files)

All tests use pytest. Run with `make test` (fast, excludes slow) or
`make test-all` (including benchmarks).

| File | Tests | Marks | What It Validates |
|------|-------|-------|-------------------|
| `test_smoke.py` | ~5 | none | All modules import, AppConfig instantiates, pipeline function exists |
| `test_cmp.py` | ~8 | none | PCHIP interpolator shape, bathtub curve monotonicity, recess bounds, sigma density-dependence |
| `test_physics_contact.py` | ~6 | none | Spectral solver convergence, bridge/conform ordering, adhesion effect, negative topology handling |
| `test_bonding_week3.py` | ~5 | none | Void risk bounds [0,1], open bond risk, motif penalty effect, physics vs. heuristic path |
| `test_anneal_week4.py` | ~5 | none | Stress index, CTE mismatch direction, gradient weight, edge stress |
| `test_yield_week5.py` | ~6 | none | Yield in [0,1], sensitivity monotonicity, seed reproducibility, edge exclusion effect |
| `test_rules_week6.py` | ~5 | none | Violation masks shape, threshold behavior, suggestions generation |
| `test_optimize_week7.py` | ~4 | slow | Fill optimizer yield improvement, budget enforcement, subtractive stress reduction |
| `test_gds_ingestion.py` | ~4 | none | GDS file parsing via gdstk, density extraction, layer selection |
| `test_cli.py` | ~5 | none | CLI command execution, output file creation, error handling |
| `test_api.py` | ~6 | none | REST endpoint HTTP status codes, response schema validation, health check |
| `test_benchmarks.py` | ~4 | slow | All 4 published-data benchmarks pass |
| `test_copper_cmp.py` | ~6 | none | Copper-specific CMP calibration, Cu recess curves, Preston equation parameters |
| `test_glass_bonding.py` | ~8 | none | Glass bonding cross-pollination, AGC/Corning/Schott material properties, bond energy prediction |
| `test_multi_node.py` | ~5 | none | Multi-node validation, distributed pipeline, cross-node consistency |
| `test_scalability.py` | ~4 | slow | Large-grid scalability, memory usage, wall-time benchmarks |
| `test_stress_adversarial.py` | ~8 | none | Edge cases: all-zero density, all-one density, tiny grids, NaN input, extreme temperatures |

### Running Tests

```bash
# Fast tests only (excludes benchmarks and GPU)
make test

# Equivalent:
pytest tests/ -q -m "not slow"

# All tests including benchmarks
make test-all

# With coverage
pytest tests/ --cov=bondability --cov-report=term-missing

# Single test file
pytest tests/test_physics_contact.py -v
```

---

## 13. Validation Benchmarks

The platform includes 4 self-validation benchmarks that compare solver
outputs against published experimental and analytical data. These run
via `bondability benchmark` or the `/api/v1/benchmarks` endpoint.

**Design principle:** NO CIRCULAR VALIDATION. Benchmarks test against
data independent of calibration inputs.

### Benchmark 1: CMP vs. Stine et al. 1998 (HOLD-OUT)

- **Reference:** Stine et al., "Rapid Characterization and Modeling of
  Pattern-Dependent Variation in CMP", IEEE TSM, 1998, Table II.
- **Method:** Default calibration has knots at {0.0, 0.3, 0.5, 0.7, 1.0}.
  Benchmark tests at HELD-OUT densities {0.1, 0.2, 0.4, 0.6, 0.8, 0.9}
  that are NOT calibration knots.
- **Checks:**
  - 6 hold-out density points within expected ranges from paper data.
  - Bathtub curve shape: minimum near 0.5 density.
  - Endpoints higher than minimum.
- **Pass criteria:** All hold-out predictions within expected ranges AND
  shape checks pass.

### Benchmark 2: Contact Solver vs. Turner & Spearing 2002

- **Reference:** Turner & Spearing, "Modeling of Direct Wafer Bonding",
  J. Appl. Phys., 2002, Fig. 4.
- **Method:** 20 nm Gaussian bump on 64x64 grid. Tests three configurations:
  thick wafer (775 um), thin wafer (10 um), high adhesion (5 J/m2).
- **Checks:**
  - Thick wafer bridges (residual gap > 0).
  - Thin wafer conforms (gap < 2 nm).
  - High adhesion reduces gap vs. standard.
  - Correct ordering: thick_gap > thin_gap.
- **Pass criteria:** All 4 qualitative physics checks pass.

### Benchmark 3: Thermal FEA vs. Suhir 1986

- **Reference:** Suhir, "Stresses in Bi-Metal Thermostats", J. Appl.
  Mech., 1986.
- **Method:** Sharp Cu/SiO2 bimetallic strip (50x50 grid, no smoothing).
  Analytical Suhir solution: sigma = E1*E2/(E1+E2) * delta_alpha * delta_T.
- **Checks:**
  - Solver converges (mandatory, no waivers).
  - Peak FD interface stress within 3x of analytical. (Why 3x? Sharp FD
    interface creates stress concentration that analytical solution averages
    out. This is CONSERVATIVE -- safe-side error.)
- **Pass criteria:** Convergence AND ratio within [0.3, 3.0].

### Benchmark 4: Yield Model Sanity + Seed Configurability

- **Method:** Uniform 40x40 die at 50% density. Tests low-defect vs.
  high-defect configurations.
- **Checks:**
  - yield(low_defect) > yield(high_defect) (monotonicity).
  - All yields in [0, 1].
  - Different random seeds produce different results (non-determinism).
- **Pass criteria:** All 3 sanity checks pass.

---

## 14. Dependencies

### Core Dependencies (required)

| Package | Version | Purpose |
|---------|---------|---------|
| numpy | >= 1.24 | Array operations, risk maps |
| scipy | >= 1.10 | Sparse solvers, PCHIP, Gaussian filter, L-BFGS-B |
| pandas | >= 2.0 | Data handling |
| pyyaml | >= 6.0 | Configuration parsing |
| pydantic | >= 2.0 | Configuration validation |
| typer | >= 0.9 | CLI framework |
| rich | >= 13.0 | CLI output formatting |
| matplotlib | >= 3.7 | Plot generation |
| shapely | >= 2.0 | Geometry operations |
| gdstk | >= 0.9.50 | GDS/OASIS layout parsing |
| scikit-learn | >= 1.3 | ML utilities |

### Optional: GPU Support (`[opt]`)

| Package | Version | Purpose |
|---------|---------|---------|
| torch | >= 2.0 | GPU thermal solver (CG), CMP GPU path |
| botorch | >= 0.9 | Bayesian optimization |

### Optional: Uncertainty Quantification (`[uq]`)

| Package | Version | Purpose |
|---------|---------|---------|
| pyro-ppl | >= 1.9 | Probabilistic programming |

### Optional: REST API (`[api]`)

| Package | Version | Purpose |
|---------|---------|---------|
| fastapi | >= 0.100 | REST API framework |
| uvicorn | >= 0.20 | ASGI server |

### Development (`[dev]`)

| Package | Version | Purpose |
|---------|---------|---------|
| pytest | >= 7.4 | Testing |
| pytest-cov | >= 4.1 | Coverage |
| ruff | >= 0.3 | Linting + formatting |
| mypy | >= 1.8 | Type checking |
| pre-commit | >= 3.5 | Pre-commit hooks |

---

## 15. Known Limitations

This section documents honest limitations for buyer due diligence.
These are engineering trade-offs, not bugs.

### 15.1 CMP Calibration Defaults

The default calibration now uses the `copper_hybrid_bonding` preset
(Enquist 2019, Kim 2022) with recess ~2.5 nm at optimal density. The
original Stine et al. 1998 aluminum curve is available as fallback.
**For production use, a fab integration team should replace with
fab-specific Cu CMP measurements.** The calibration module
(`bondability.cmp.calibration`) supports loading custom fab data.

### 15.2 Linear Elastic Thermal Solver (No Plasticity)

The thermal FEA solver computes linear elastic stress. At 300C anneal
with sharp Cu/SiO2 interfaces, peak von Mises stress can exceed Cu yield
(~300 MPa at temperature). In reality, copper yields plastically and
stress redistributes. Implementing proper incremental plasticity (load
stepping, tangent stiffness, deviatoric radial return) was a scope
decision. The solver reports honest elastic stress and is therefore
CONSERVATIVE (overestimates stress at interfaces). For a more accurate
stress field, use `smooth_interface=True` to eliminate the FD singularity.

### 15.3 Contact Solver Under-Resolution at Tile Scale

The Dugdale cohesive zone width (dc = 5 nm) is resolved at nanometer
scale, but the tile grid is at 25 um. At tile resolution, the solver
gives qualitatively correct contact/bridging behavior but quantitative
gap values are approximate (2-5x error). Use `refine_factor >= 4` for
production accuracy (subdivides each tile into 4x4 elements, 16x slower).

### 15.4 Correlation Length is the Dominant Yield Knob

The yield model's `correlation_length_um` parameter controls the number
of independent failure clusters and dominates the final yield number.
The default (500 um) is a reasonable starting point but is NOT calibrated
to any specific fab's defect correlation data. A buyer must calibrate
this against actual fab yield data. Changing this parameter from 200 um
to 2000 um can swing yield by 20+ percentage points.

### 15.5 No 3D CMP Model

The CMP model is 2D (single-layer effective density). It does not model
multi-layer CMP interactions, dishing propagation through stacked layers,
or through-silicon via (TSV) topography effects on CMP.

### 15.6 No Wafer-Level Warpage Model

The bonding model uses a simple `warpage_um` parameter but does not
compute wafer warpage from first principles (thermal gradient, film
stress, CTE mismatch at wafer scale). A proper warpage model would
require wafer-scale FEA with film stack integration.

### 15.7 Heuristic Motif Detection

Motif detection (isolated pads, TSV proximity, extreme gradients) uses
threshold-based heuristics on the density map, not true pattern recognition.
It does not detect specific layout structures (dummy fill, guard rings,
seal rings).

### 15.8 ML Surrogate Is Experimental

The `bondability.ml` module (surrogate model and training pipeline) is
present but experimental. It is NOT used in the production pipeline. The
archived ML dataset (200 samples in archive/ml_datasets/) is small and
is not a trained production model.

### 15.9 REST API Job Store Is In-Memory

The FastAPI job store is in-memory (Python dict). Jobs are lost on server
restart. For production deployment, replace with a persistent job store
(Redis, database).

### 15.10 Single-Threaded Pipeline

The pipeline runs stages sequentially on a single thread. No parallel
execution of independent stages. The GPU thermal solver uses GPU
parallelism internally but the overall pipeline is single-threaded.

---

## 16. Buyer Due Diligence Checklist

### Code Quality

- [ ] Run `make test` -- all 17 test files pass
- [ ] Run `make test-all` -- all tests including slow benchmarks pass
- [ ] Run `bondability benchmark` -- all 4 published-data benchmarks pass
- [ ] Run `make lint` -- ruff reports clean
- [ ] Run `make typecheck` -- mypy reports clean (with ignore_missing_imports)
- [ ] Verify `pip install -e .` succeeds with Python >= 3.10
- [ ] Verify `pip install -e ".[all]"` installs all optional dependencies

### Physics Validation

- [ ] CMP benchmark: hold-out predictions within Stine 1998 ranges
- [ ] Contact benchmark: thick bridges, thin conforms (Turner 2002)
- [ ] Thermal benchmark: FD/analytical ratio within [0.3, 3.0] (Suhir 1986)
- [ ] Yield benchmark: monotonic, bounded, seed-sensitive
- [ ] Run `bondability benchmark --verbose` and inspect per-check results
- [ ] Verify CMP bathtub shape with `cmp_sanity_sweep()` -- minimum near d=0.5

### End-to-End Pipeline

- [ ] Run `bondability run --input examples/demo.npy --output test_results/`
- [ ] Verify output directory contains: report.html, yield_summary.json,
      rules_summary.json, risk_maps/, plots/, violations/
- [ ] Open report.html -- verify executive summary, risk maps, yield
      distribution, sensitivity analysis render correctly
- [ ] Verify .lyrdb file opens in KLayout

### REST API

- [ ] Run `bondability serve` and visit http://localhost:8000/docs
- [ ] POST /api/v1/simulate with a 10x10 density array
- [ ] GET /api/v1/simulate/{job_id} -- verify completion
- [ ] GET /api/v1/health -- verify solver availability
- [ ] GET /api/v1/benchmarks -- verify all pass

### Optimizer

- [ ] Run `bondability optimize --input examples/demo.npy --output opt_test/`
- [ ] Verify optimized_layout.npy and report.json are produced
- [ ] Verify report.json contains yield, void_risk, delam_risk, gpu_peak_MPa

### Configuration

- [ ] Run `bondability config show` -- verify JSON output
- [ ] Run `bondability config validate configs/process_example.yaml`
- [ ] Modify configs/process_example.yaml and re-validate

### Archive Provenance

- [ ] Verify archive/runs/ contains 6 complete run directories
- [ ] Verify archive/out_physics_final/ contains report.html + risk maps
- [ ] Verify archive/scripts/ contains 45 historical proof scripts
- [ ] Note: archive scripts are NOT production code and may have stale imports

### Calibration Assessment

- [ ] Review default CMP calibration (now `copper_hybrid_bonding` preset) --
      consider replacing with fab-specific Cu CMP data for production
- [ ] Review correlation_length_um (500 um default) -- understand this is
      the single most impactful tunable and must be calibrated
- [ ] Review anneal temperature (300C default) -- verify against target
      process
- [ ] Review overlay sigma (20 nm default) -- verify against target
      process capability

### IP and Scope

- [ ] Production code is ~61 Python files, ~13,000 LOC in bondability/
- [ ] 45 archive scripts are historical proofs, not production code
- [ ] ML module is experimental and not in production pipeline
- [ ] No external API keys, no cloud dependencies, no telemetry
- [ ] MIT license

---

## References

- Stine et al., "Rapid Characterization and Modeling of Pattern-Dependent
  Variation in CMP", IEEE TSM, 1998
- Turner & Spearing, "Modeling of Direct Wafer Bonding", J. Appl. Phys., 2002
- Suhir, "Stresses in Bi-Metal Thermostats", J. Appl. Mech., 1986
- Murphy, "Yield, Reliability, and Defect Density", Proc. IEEE, 1964
- Stapper, "Modeling of Defects in ICs", IBM J. Res. Dev., 1983
- Cunningham, "The Use and Evaluation of Yield Models", IEEE TSM, 1990
- Persson, "Contact Mechanics for Randomly Rough Surfaces", Surf. Sci. Rep., 2006
- Hutchinson & Suo, "Mixed Mode Cracking in Layered Materials", Adv. Appl. Mech., 1992
- Maugis, "Adhesion of Spheres: The JKR-DMT Transition", J. Colloid
  Interface Sci., 1992
- Ouma et al., "Characterization and Modeling of Oxide CMP", IEEE TSM, 2002
- Timoshenko & Goodier, "Theory of Elasticity", 3rd Ed., 1970
- Greenwood & Williamson, "Contact of Nominally Flat Surfaces", Proc. R.
  Soc. London A, 1966
