# Recommended Tools & Integrations — DECISIONS LOCKED

> Companion file to `polymer_gnn_1_year_execution_plan.md` (§0).
> **Decisions finalised:** Sep 13, 2026.

---

## ✅ Selected & Integrated

| Category | Tool | Status | Where |
|----------|------|--------|-------|
| Experiment Tracking | **W&B Free Tier** | ✅ Integrated | `src/training/trainer.py` — pass `wandb_enabled=True` |
| HPO | **Optuna** | ✅ Integrated | `scripts/hpo_optuna.py` — Bayesian search, saves best to YAML |
| GNN Library | **PyTorch Geometric** | ✅ Already in use | `src/models/`, `src/data/` |
| Equivariant GNN | **e3nn** | ✅ In `environment.yml` | Install ready for Month 10 EGNN |
| Simulator | **NumPy (custom)** | ✅ Built | `src/simulator/numpy_simulator.py` |
| Data Storage | **NPZ (NumPy)** | ✅ In use | `data/raw/*.npz` |
| Data Sharing | **Google Drive** | ✅ Decided | Zip and share link — zero setup |
| Compute (primary) | **Kaggle Kernels** | ✅ Decided | Free 30 GPU h/week |
| Compute (later) | **University HPC** | ⏳ Upgrade path | For Month 9–11 sweeps if needed |
| Plotting (static) | **Matplotlib + Seaborn** | ✅ Integrated | `src/evaluation/plotting.py` |
| Plotting (interactive) | **Plotly** | ✅ Integrated | `src/evaluation/plotting.py` |
| Plotting (3D) | **py3Dmol** | ✅ In `environment.yml` | For chain structure visualisation |
| Report | **LaTeX + Overleaf** | ✅ Decided | Month 12 |
| Local fallback viewer | **TensorBoard** | ✅ In `environment.yml` | Backup for offline viewing |

---

## ❌ Not Selected (and why)

| Tool | Reason |
|------|--------|
| MLflow | Unnecessary with W&B; more setup for less features |
| Ray Tune | Overkill for Kaggle — Optuna is simpler |
| Ax / BoTorch | Complex API; skip unless Optuna is insufficient |
| DGL | Not needed alongside PyG |
| MACE | e3nn is more flexible for custom architectures |
| HOOMD-blue | NumPy simulator is sufficient for our scale |
| LAMMPS | Overkill for single-chain N=30–200 |
| OpenMM | Unnecessary alongside custom NumPy simulator |
| HDF5 | NPZ is fine until data > 10 GB |
| Git LFS | Google Drive is simpler for team sharing |
| Google Colab Pro | Kaggle is free; upgrade path exists to Uni HPC |
| Lightning.ai | Kaggle is sufficient |
| Quarto | LaTeX + Overleaf is standard for academic reports |
| TensorFlow/Keras | Stick with PyTorch |

---

## How to Use the Integrations

### W&B Experiment Tracking
```python
# In your training script:
trainer = Trainer(
    model=model,
    train_loader=train_loader,
    val_loader=val_loader,
    config=config,
    wandb_enabled=True,           # ← toggle this
    wandb_project='polymer-gnn',
    wandb_run_name='baseline-v1',
)
history = trainer.train()
# → loss curves, LR, gradients, and best model auto-logged to wandb.ai
```

### Optuna HPO
```bash
# Quick test (5 trials, 10 epochs each):
python scripts/hpo_optuna.py --n-trials 5 --epochs 10

# Full search with W&B:
python scripts/hpo_optuna.py --n-trials 50 --wandb

# Resume from SQLite:
python scripts/hpo_optuna.py --study-name my-study --storage sqlite:///hpo.db
```

### Plotting
```python
from src.evaluation.plotting import (
    plot_training_curves,      # train/val loss
    plot_rollout_msd,          # g1/g2/g3 log-log
    plot_rouse_spectrum,       # tau_p vs p
    plot_bond_length_histogram,# bond length distribution
    plot_model_comparison,     # grouped bar chart
    plot_hpo_importance,       # Optuna importance
    plot_rollout_stability,    # metric drift over rollout
)
```

---

## Key Papers (for reference)

| Paper | Why |
|-------|-----|
| Sanchez-Gonzalez et al. (2020) | GNS baseline — training-time noise injection (Month 9) |
| Kremer & Grest (1990) J. Chem. Phys. 92, 5057 | The KG model — our exact setup |
| Pfaff et al. (2021) ICLR | Rollout stability tricks |
| Batatia et al. (2022) MACE | Equivariant architecture (Month 10) |
| de Gennes (1979) *Scaling Concepts* | Rouse model, scaling, dynamics |
| Gedde (1999) *Polymer Physics* | Molten state, chain conformations |
