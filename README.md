# Automated 3D Missile Aerodynamic Database & Surrogate Pipeline

## 1. Project Overview & Tech Stack
An end-to-end automated Computational Fluid Dynamics (CFD) and Machine Learning pipeline for a slender cruciform missile (L = 1200 mm, D = 100 mm, boat-tail \(D_b = 80\text{ mm}\)).

* **CAD & Meshing:** Cadence Fidelity Pointwise (Hybrid T-Rex boundary layer + isotropic farfield, native `.su2` export)
* **CFD Solver:** SU2 (Compressible RANS / Euler, executed via MPI across 4 physical CPU cores)
* **Post-Processing:** ParaView (`pvbatch --force-offscreen-rendering` automated rendering)
* **Automation & ML:** Python (`subprocess`, `scipy.stats.qmc`, `pandas`, `scikit-learn` / `PyTorch`)
* **Hardware Target:** Intel Core i5-10210U (4 Physical Cores), WSL2 Linux environment

---

## 2. Repository Structure

```text
missile-aero-surrogate/
├── phase1_sandbox/
│   ├── 01_supersonic_wedge_schemes/   # Roe vs. AUSM+ & Venkatakrishnan limiter tests
│   ├── 02_pointwise_trex_tests/       # T-Rex boundary layer & y+ verification
│   └── 03_axisymmetric_body_gci/      # Grid Convergence Index (GCI) verification
├── cad_and_mesh/                      # Clean 3D missile .STEP & Pointwise Glyph scripts
├── su2_baseline/                      # Validated 3D SU2 .cfg & baseline history logs
├── python_scripts/
│   ├── doe_generator.py               # Latin Hypercube Sampling (Mach, AoA)
│   ├── run_pipeline.py                # Master MPI solver & parser orchestrator
│   └── post_process.py                # Headless pvbatch slice & contour exporter
├── surrogate_model/                   # Trained ML surrogate & accuracy validation plots
├── docs/images/                       # CAD renders, mesh quality, & Mach contours
├── README.md                          # Project overview & workflow
├── ENGINEERING_LOG.md                 # Dated lab notebook of solver & mesh decisions
└── .gitignore                         # Blocks large binary meshes/solutions (*.su2, *.vtu)
```

---

## 3. Two-Phase Execution Roadmap

### Phase 1: Foundations (Meshing, Schemes & Script Prototyping)
* **Step 1.1 — Pointwise Meshing Mastery:**
  * Import `.STEP` geometries, build watertight **Models/Quilts**, and dimension **Connectors**.
  * Master **T-Rex** anisotropic prism extrusion for target $y^+$ spacing and verify cell skewness/orthogonality via Pointwise **Examine**.
* **Step 1.2 — SU2 Numerical Scheme Verification:**
  * Benchmark **Roe** vs. **AUSM+** convective schemes and **Venkatakrishnan** slope limiters on a 2D supersonic wedge against analytical oblique shock relations.
  * Test **CFL ramping** (`CFL_ADAPT`) and **Spalart-Allmaras (SA) / Menter SST** turbulence models on an axisymmetric fuselage with boat-tail wake separation.
* **Step 1.3 — Headless Automation Prototyping:**
  * Record a ParaView visualization pipeline using **Start/Stop Trace** and run it headlessly via `pvbatch --force-offscreen-rendering`.
  * Build a prototype Python wrapper to modify `.cfg` files, launch `mpirun -n 4 SU2_CFD`, and parse `history.csv`.

### Phase 2: Full 3D Missile Execution & Surrogate Modeling
* **Step 2.1 — Symmetric Bullet-Domain 3D Mesh:**
  * Cut the 1200 mm missile along its vertical symmetry plane ($\beta = 0^\circ$) to halve cell count (~400k–600k cells).
  * Construct a **bullet-shaped farfield domain** ($2L$ upstream, $12L$ downstream, $5L$ radial) with conical shock/wake refinement sources in Pointwise.
* **Step 2.2 — Baseline 3D Validation:**
  * Converge a baseline 3D operating point (e.g., Mach 2.0, $\alpha = 4^\circ$) and verify normal force ($C_N$), axial force ($C_A$), and pitching moment ($C_m$) trends against slender-body/DATCOM correlations.
* **Step 2.3 — Automated Parametric Sweep:**
  * Generate 30–50 `(Mach, AoA)` operating points using **Latin Hypercube Sampling (LHS)**.
  * Execute `run_pipeline.py` to solve cases automatically, log coefficients into `aero_database.csv`, and export Mach/pressure PNGs via `pvbatch`.
* **Step 2.4 — Surrogate Model Training:**
  * Train a **Gaussian Process Regressor (GPR)** or **Neural Network** on 80% of `aero_database.csv` and validate predictions ($R^2$ and MAPE) on the remaining 20%.
