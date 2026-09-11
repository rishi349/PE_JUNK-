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

- [ ] **Implement Physics Enhancements (from coding_agent_brief.md)**
  - [ ] Task 1: `src/evaluation/rouse_modes.py` — Rouse mode analysis
  - [ ] Task 2: `src/evaluation/msd.py` — g₁, g₂, g₃ MSD subdiffusion functions
  - [ ] Task 3: `src/evaluation/scaling.py` — Finite-size ν_eff for scaling study
  - [ ] Task 4: `src/training/losses.py` — Rouse mode consistency loss (ablation only)
  - [ ] Task 5: `src/evaluation/temperature_check.py` — FDT temperature check
  - [ ] Task 6: Report text — Rouse/Zimm/solvent-quality framing (Month 12)
  - [ ] Task 7: `src/evaluation/rollout.py` — Rouse mode spectrum rollout eval metric
