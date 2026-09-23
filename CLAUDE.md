# CLAUDE.md — Compact Project Knowledge Summary

> Paste this into any fresh AI chat to restore full project context instantly.
> Last updated: Sep 23, 2026

---

## What We're Building

**Physics-Informed Graph Neural Network (GNN) for Polymer Chain Dynamics**

A GNN trained to predict the next-step displacement of beads in a coarse-grained single polymer chain (Kremer-Grest model), trained under matched conditions against a plain message-passing GNN baseline, compared on long-rollout stability.

---

## Team & Timeline

- **Team:** 2–3 people
- **GPU:** Shared/limited Colab Pro
- **Timeline:** 12 months (52 weeks), started ~May 2026
- **Current status:** End of Month 4 (Sep 2026) — code complete, production data generated & validated
- **Git repos:**
  - Code → `github.com/rishi349/PINN`
  - Logs → `github.com/rishi349/PE_JUNK-`

---

## Locked Decisions (Project Contract)

| Decision | Value | Locked |
|---|---|---|
| Dynamics regime | Overdamped Langevin (Brownian) | ✅ |
| Integrator | Euler–Maruyama | ✅ |
| Bond potential | Harmonic (→ FENE for production) | ✅ |
| Excluded volume | WCA (repulsive-only LJ) | ✅ |
| Prediction target | Next-step displacement Δr | ✅ |
| Data split | 70/15/15 by trajectory (no leakage) | ✅ |
| Chain length (main) | N = 30 | ✅ |
| OOD arms | N = 50, 100, 200 | ✅ |
| Units | Reduced LJ (ε=σ=m=1, k_BT=1) | ✅ |

---

## Physics Equations

**Overdamped Langevin:**
```
γ dr/dt = F(r) + ξ(t),   ⟨ξ(t)ξ(t')⟩ = 2γk_BT δ(t-t')
```

**Euler–Maruyama integrator:**
```
r_new = r + (dt/γ) * F + sqrt(2 * k_BT * dt / γ) * N(0,1)
```

**FENE bond:**
```
U_FENE = -0.5 * k * R0² * ln(1 - (r/R0)²),   k=30, R0=1.5σ
```

**WCA (excluded volume):**
```
U_WCA = 4ε[(σ/r)¹² - (σ/r)⁶] + ε   for r < 2^(1/6)σ
```

---

## Repository Structure

```
BASHI+OK/
├── PINN/                          ← Code repo (github.com/rishi349/PINN)
│   ├── src/
│   │   ├── simulator/             ← NumpySimulator + HoomdSimulator (GPU fallback)
│   │   ├── physics/               ← forces.py, integrators.py
│   │   ├── data/                  ← graph_builder, dataset, normalization
│   │   ├── models/                ← baseline_gnn.py, naive_baselines.py
│   │   ├── training/              ← losses.py, trainer.py
│   │   └── evaluation/            ← metrics, rollout, rouse_modes, msd, temperature_check, plotting
│   ├── scripts/
│   │   ├── generate_data.py, generate_all_arms.py, train.py, evaluate.py
│   │   ├── diagnose_equilibration.py, verify_production.py, verify_remaining.py
│   │   ├── visualize.py, visualize_trajectory.py
│   │   ├── plot_rg2_vs_N.py       ← Flory scaling analysis
│   │   ├── plot_force_extension.py ← Force vs extension analysis
│   │   └── plot_msd.py            ← MSD vs time (from existing trajectory)
│   ├── tests/                     ← 152 passing tests
│   └── configs/
│       ├── default.yaml           ← Full production config (3.7M steps)
│       └── short.yaml             ← Quick-test config (100× fewer steps)
├── professor/                     ← Professor workspace (github.com/SiddharthaReddy018/professor)
│   ├── PINN/                      ← Subset of main PINN (simulator + analysis, no GNN/training)
│   │   ├── data/raw/trajectory_0000.json  ← 50MB production trajectory
│   │   └── plots/                 ← Generated GIFs, analysis plots
│   ├── activity_log-professor.md
│   └── *.ipynb                    ← Kaggle notebooks (not in main PINN)
├── logs/                          ← Logs repo (github.com/rishi349/PE_JUNK-)
│   ├── activity_log.md
│   ├── task.md
│   ├── walkthrough.md
│   ├── CLAUDE.md                  ← This file
│   ├── recommended_tools_and_integrations.md
│   └── books/
│       ├── de_gennes_key_concepts.md
│       └── gedde_key_concepts.md
├── plan docs/                     ← Read-only reference (no git)
│   └── polymer_gnn_1_year_execution_plan.md
└── .agents/rules/workspace_rules.md

---

## Current Code State (Sep 23, 2026)

### What's Done ✅
- Full simulator (NumPy, Euler-Maruyama, WCA+harmonic) with HoomdSimulator (GPU fallback)
- Physics fixes: 3-layer safety clamp (WCA clamp, force cap 1000ε/σ, displacement cap)
- Production trajectory generated & validated: `trajectory_0000.json` (50MB, 10k frames, N=30)
- Graph construction → PyTorch Geometric dataset → normalization
- Baseline GNN architecture (message-passing, 2–4 layers)
- Trainer with checkpoint save/load + W&B integration
- Full evaluation suite: rollout, bond-length, Rg, MSD, Rouse modes, FDT temperature check
- Naive baselines: ZeroDisplacementBaseline (0a) and GlobalStatsBaseline (0b)
- Automated split-leakage assertion
- Bond-length penalty + excluded-volume penalty losses (for Month 7 physics-informed model)
- Multi-arm data generation script (N=30/50/100/200)
- HPO via Optuna (`scripts/hpo_optuna.py`)
- 3 analysis scripts: Rg² vs N, force-extension, MSD vs time
- 5 visualization GIFs (downsampled + continuous modes)
- 152 unit tests, 0 failures

### What's Pending ⏳
- Generate remaining trajectory arms (N=50, 100, 200) for OOD evaluation
- First real training run → `checkpoints/best.pt`
- Baseline evaluation plots (Month 5–6)

---

## Month-by-Month Reference (§9 of plan)

| Month | Focus | Gate |
|---|---|---|
| 1 | Physics study + environment setup | ✅ Done |
| 2 | Physics deep-dive + simulator starts | ✅ Done |
| 3 | Full simulator + pilot validation | ✅ Done |
| **4** | **Production dataset + naive baselines** | **✅ Done (code + data + validation)** |
| 5 | Baseline GNN training | ⏳ Waiting for data |
| 6 | Rollout evaluation + HPO | ⏳ |
| 7 | Physics-informed model | ⏳ |
| 8 | Controlled comparison + data efficiency | ⏳ |
| 9 | Ablation matrix + OOD grid | ⏳ |
| 10 | EGNN + scaling study | ⏳ |
| 11 | Momentum-conserving + UQ + interpretability | ⏳ |
| 12 | Final consolidation + report | ⏳ |

---

## Key Physics Checks (all passing ✅)

- Bond length histogram centred at ~1.06σ (harmonic+WCA equilibrium)
- Temperature stable at k_BT = 1.0 (FDT check within 5%)
- No NaN/Inf positions in production (0 extreme bonds, 0 extreme forces)
- R_g ~ N^ν with ν ≈ 0.584 (expected good-solvent SAW: 0.588, within 0.7%)
- WCA clamp: 0 activations; force cap: 48/1,000,000 steps (0.005%)
- Autocorrelation time τ_int(Rg) = 34.2τ → ~15 independent samples per trajectory
- MSD validation: g1 exponent = 0.621 (Rouse crossover ✓), g3 exponent = 0.917 (diffusive ✓)
- Force-extension: monotonic increase 1.37 → 6.52 ε/σ from 52%→95% L_c (FJC/WLC consistent)

---

## Models in the Plan

| ID | Model | Month |
|---|---|---|
| 0a | Zero-displacement baseline | ✅ Done |
| 0b | Global-stats random draw | ✅ Done |
| 1 | Baseline message-passing GNN | Month 5–6 |
| 3 | Physics-informed (+ bond + EV loss) | Month 7 |
| 4 | Baseline + noise injection | Month 9 ablation |
| 5 | EGNN (equivariant) | Month 10 |
| 6 | Momentum-conserving GNN | Month 11 |

---

## Key Files to Read First (in order)

1. `plan docs/polymer_gnn_1_year_execution_plan.md` — master reference
2. `PINN/project_contract.md` — locked decisions
3. `logs/task.md` — current TODO state
4. `logs/activity_log.md` — full history of what was done
