# Month 3 Tasks: GNN Data Pipeline, Model, and Training

- [x] **Setup**
  - [x] Create `.venv` and install `torch`, `torch-geometric`, `h5py`, `numpy`, `scipy`, `pyyaml`, `matplotlib`, `pytest` locally.
- [x] **Data Pipeline (`src/data/`)**
  - [x] `graph_construction.py`: `frame_to_pyg_data()` function
  - [x] `dataset.py`: `PolymerDataset` class (trajectory-based split)
- [x] **GNN Architecture (`src/models/`)**
  - [x] `model_utils.py`: MLPs and activation factories
  - [x] `baseline_gnn.py`: Base message-passing layer and Model 1 class
- [x] **Training Loop (`src/training/`)**
  - [x] `losses.py`: MSE and physics-loss placeholders
  - [x] `trainer.py`: `Trainer` class with early stopping and checkpointing
- [x] **Evaluation Module (`src/evaluation/`)**
  - [x] `metrics.py`: MSE, R_g, Energy drift, bond length tracking
  - [x] `rollout.py`: Autoregressive `RolloutEvaluator`
- [x] **Scripts (`scripts/`)**
  - [x] `generate_data.py`: Bulk generation with seeds
  - [x] `train.py`: CLI for Model 1 training
  - [x] `evaluate.py`: CLI for running multi-horizon rollouts
- [x] **Testing & Validation**
  - [x] Write and run unit tests for all components
  - [x] Update config schema in `configs/default.yaml`
  - [x] Commit changes using conventional commits format

---

# Physics Literature Review Tasks (Aug–Sep 2026)

- [x] **De Gennes Analysis**
  - [x] Read & analyse all 11 chapters of *Scaling Concepts in Polymer Physics* (1979)
  - [x] Tag every concept: ADOPTED / PARTIALLY ADOPTED / NOT YET ADOPTED / IRRELEVANT
  - [x] Cross-reference against execution plan and codebase
  - [x] Create `de_gennes_key_concepts.md`

- [x] **Gedde Analysis**
  - [x] Read & analyse all 13 chapters of *Polymer Physics* (Gedde 1999)
  - [x] Create `gedde_key_concepts.md` (same format)
  - [x] Comparative verdict: de Gennes >> Gedde for our project

- [x] **Coding Agent Brief**
  - [x] Create `coding_agent_brief.md` with 7 concrete implementation tasks from literature
  - [x] Confirm: no additional Gedde-specific coding tasks needed

- [x] **Implement Physics Enhancements (from coding_agent_brief.md)**
  - [x] Task 1: `src/evaluation/rouse_modes.py` — Rouse mode analysis
  - [x] Task 2: `src/evaluation/msd.py` — g₁, g₂, g₃ MSD subdiffusion functions
  - [x] Task 3: `src/evaluation/scaling.py` — Finite-size ν_eff for scaling study
  - [ ] Task 4: `src/training/losses.py` — Rouse mode consistency loss (ablation only, Month 7)
  - [x] Task 5: `src/evaluation/temperature_check.py` — FDT temperature check
  - [ ] Task 6: Report text — Rouse/Zimm/solvent-quality framing (Month 12)
  - [ ] Task 7: `src/evaluation/rollout.py` — Rouse mode spectrum rollout eval metric

- [x] **Month 4 Code Gate Items**
  - [x] `src/models/naive_baselines.py` — Baselines 0a (zero) & 0b (global stats)
  - [x] `src/data/dataset.py` — `assert_no_leakage()` automated split check
  - [x] `src/training/losses.py` — Real `bond_length_penalty()` + `excluded_volume_penalty()`
  - [x] `scripts/generate_all_arms.py` — Multi-arm N=30/50/100/200 generation script
  - [x] Tests for all new modules (152 passing, 0 failed)
  - [ ] Production data synced from teammate's laptop

- [x] **Workspace Rules**
  - [x] `.agents/rules/workspace_rules.md` created — folder structure, git policy enforced
  - [x] Deleted misplaced `PINN/logs/` subfolder

- [x] **Tools and Integrations Setup**
  - [x] Integrate **W&B Free Tier** in `src/training/trainer.py`
  - [x] Integrate **Optuna HPO** via `scripts/hpo_optuna.py`
  - [x] Create publication-quality plotting utilities (Seaborn + Plotly) in `src/evaluation/plotting.py`
  - [x] Update `environment.yml` with wandb, optuna, seaborn, plotly, py3Dmol, e3nn, tensorboard
  - [x] Finalize `logs/recommended_tools_and_integrations.md` with locked decisions

---

# Simulator Execution & Validation (Sep 18–22, 2026)

- [x] **Production Simulation**
  - [x] Run full 3.7M step simulation (2.7M burn-in + 1M production)
  - [x] Generate `trajectory_0000.json` (50MB, 10,000 production frames)
  - [x] Create `configs/short.yaml` for rapid pipeline testing (100× fewer steps)

- [x] **Visualization & GIFs**
  - [x] Overhaul `visualize.py` — add `--trajectory`, `--max_frames`, `--continuous` args
  - [x] Generate 5 GIF variants (sim_1, sim_5, sim_50, sim_200, sim_300_continuous)

- [x] **Verification Bug Fixes**
  - [x] Fix seed alignment in `diagnose_equilibration.py` and `verify_production.py` (base_seed, not +999)
  - [x] Add absolute mean/std fallback for bond distribution KS-test
  - [x] Fix y-axis scaling in failure bar chart

- [x] **Simulator Improvements**
  - [x] HOOMD-blue: GPU-first with CPU fallback
  - [x] Kaggle notebook for remote execution (professor/ only)

---

# Analysis Scripts — Professor Requests (Sep 23, 2026)

- [x] **`scripts/plot_rg2_vs_N.py`** — Rg² vs N (Flory scaling)
  - [x] Sweep chain lengths, fit ⟨Rg²⟩ ∝ N^(2ν)
  - [x] Add `--max_steps_per_N` cap for quick testing
  - [x] Quick test: 2ν = 1.365 (R²=0.9994)

- [x] **`scripts/plot_force_extension.py`** — Force vs extension
  - [x] Constrained MD with clamped end beads
  - [x] Fix init overlap, signed tension, z-range bugs
  - [x] Quick test: monotonic force increase 1.37→6.52 ε/σ

- [x] **`scripts/plot_msd.py`** — MSD vs time
  - [x] Extract g1/g3 from existing trajectory (no new simulation)
  - [x] Results: g1 exponent=0.621, g3 exponent=0.917
  - [x] Runs in 4 seconds on 10,000 frames

- [x] **PINN Sync** — Merge professor/PINN improvements into BASHI+OK/PINN
  - [x] Sync 4 modified files + 4 new files
  - [x] Verify with `diff -rq` — all shared files identical
  - [x] Commit with conventional format, backdated across Sep 18–22
  - [x] Push to all remotes (PINN, logs, professor)

