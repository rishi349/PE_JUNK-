# Recommended Tools & Integrations

> Companion file to `polymer_gnn_1_year_execution_plan.md` (§0).
> Lists tools found during research that are **not** required by the plan but worth a team discussion before committing to any of them.
> Updated: Sep 2026

---

## Experiment Tracking

| Tool | Purpose | Notes |
|------|---------|-------|
| **Weights & Biases (W&B)** | Track runs, hyperparams, metrics, plots | Free tier. Recommended — set up in Month 1 before any training. W&B `wandb.log()` is a one-liner to add. |
| MLflow | Alternative to W&B, fully local | Good if you don't want cloud upload. More setup. |
| TensorBoard | Lightweight, built into PyTorch | Fine for quick loss curves; lacks the run-comparison features of W&B. |

**Recommendation:** W&B free tier. Create account now at wandb.ai — retrofitting to 50 runs in Month 6 is painful.

---

## Hyperparameter Optimization (HPO)

| Tool | Purpose | Notes |
|------|---------|-------|
| **Optuna** | Bayesian HPO, tree-structured Parzen estimator | `pip install optuna`. Used in Month 6. Integrates with W&B. |
| Ray Tune | Distributed HPO | Overkill for Colab; Optuna is simpler. |
| Ax / BoTorch | Bayesian optimization from Facebook | More complex API; skip unless Optuna proves insufficient. |

**Recommendation:** Optuna. Simple API, good docs, free.

---

## GNN Libraries

| Tool | Purpose | Notes |
|------|---------|-------|
| **PyTorch Geometric (PyG)** | Graph neural networks | Already in use. `torch_geometric` |
| DGL | Alternative GNN library | Not needed alongside PyG. |
| e3nn | Equivariant neural networks | Needed for Month 10 EGNN. Install separately: `pip install e3nn` |
| MACE | Equivariant message-passing | Alternatively usable for equivariant baseline |

---

## Simulation / MD Tools (Reference)

| Tool | Purpose | Notes |
|------|---------|-------|
| **NumPy simulator** | Custom overdamped Langevin | ✅ Already built — `src/simulator/numpy_simulator.py` |
| HOOMD-blue | GPU-accelerated MD | Optional for large-scale data generation if Colab GPU is available. `hoomd_simulator.py` stub exists. |
| LAMMPS | General MD, supports KG model | CPU/GPU. Overkill unless you need thousands of long trajectories. |
| OpenMM | Python-friendly MD | Good Python API. Alternative to HOOMD if needed. |

---

## Data Storage

| Tool | Purpose | Notes |
|------|---------|-------|
| **NPZ (NumPy)** | Compact trajectory storage | ✅ Currently using. Default for `generate_data.py` |
| HDF5 / h5py | Better for very large datasets | Consider switching if data/raw > 10 GB |
| **Git LFS** | Large files in git | Use `git lfs track "data/raw/*.npz"` to share data via GitHub |
| Google Drive | Simplest team data sharing | Zero setup; just zip and share the link |

**Current recommendation:** Share data via Google Drive zip until you have >10 GB.

---

## Compute (GPU)

| Platform | Notes |
|----------|-------|
| **Google Colab Pro** | ~$10–50/month. Recommended for Months 5–11. |
| Kaggle Kernels | Free 30 GPU h/week. Good backup. |
| Lightning.ai | Free tier with GPU, good PyTorch integration |
| University HPC | Check with supervisor — if available, use this for Month 9–11 sweeps |

**Budget rule from plan:** Spend almost nothing on GPU during Months 1–4 (CPU simulation only). Concentrate budget in Months 5–11 for training.

---

## Plotting & Visualization

| Tool | Notes |
|------|-------|
| **Matplotlib** | ✅ Already used in scripts/visualize.py |
| Seaborn | Prettier statistical plots. Worth adding for report figures. |
| Plotly | Interactive. Good for rollout visualization. |
| py3Dmol | 3D chain structure visualization |

---

## Report Writing

| Tool | Notes |
|------|-------|
| LaTeX + Overleaf | ✅ Recommended for Month 12 report |
| Quarto | Notebook-to-paper pipeline |
| arXiv | Target venue for final report (physics/ML workshop or preprint) |

---

## Key Papers to Read (for the team)

| Paper | Why |
|-------|-----|
| Sanchez-Gonzalez et al. (2020) "Learning to Simulate Complex Physics with Graph Networks" | The GNS baseline — training-time noise injection (ablation row Month 9) |
| Kremer & Grest (1990) J. Chem. Phys. 92, 5057 | The KG model — our exact simulation setup |
| Pfaff et al. (2021) "Learning Mesh-Based Simulation with Graph Networks" | ICLR 2021. Rollout stability tricks. |
| Batatia et al. (2022) MACE | Equivariant architecture (Month 10) |
| de Gennes (1979) *Scaling Concepts in Polymer Physics* | **Read Ch. IV, VI, VII** — Rouse model, scaling, dynamics |
| Gedde (1999) *Polymer Physics* | Ch. 6 (molten state), Ch. 2 (chain conformations) |

---

## Not Recommended / Skip

- **TensorFlow/Keras** — Stick with PyTorch throughout
- **HOOMD in Month 1–4** — NumPy simulator is sufficient and easier to debug
- **Full reptation/entanglement models** — Out of scope (single chain, no entanglements at N=30)
- **Shear stress / rheology** — Not in scope (no applied flow field, equilibrium only)
