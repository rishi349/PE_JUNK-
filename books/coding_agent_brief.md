# Coding Agent Brief: Physics Enhancements from De Gennes
## For use alongside the codebase and 1-year execution plan

> **Purpose:** This document is designed to be given to a coding agent along with:
> 1. The current codebase at [PINN/](file:///Users/k.siddharthareddy/Documents/BASHI+OK/PINN)
> 2. The [1-year execution plan](file:///Users/k.siddharthareddy/Documents/BASHI+OK/polymer_gnn_1_year_execution_plan.md)
> 3. The [de Gennes key concepts analysis](file:///Users/k.siddharthareddy/.gemini/antigravity-ide/brain/a6e22f8a-0c06-4ec7-a141-734c79b516b6/de_gennes_key_concepts.md)
>
> It contains **concrete implementation tasks** with exact equations, file locations, and test criteria. All physics comes from de Gennes' *Scaling Concepts in Polymer Physics* (1979), cross-validated against Kremer & Grest (1990), Doi & Edwards (1986), and Rubinstein & Colby (2003).

---

## Context for the Coding Agent

You are working on a Physics-Informed GNN project that simulates and learns the dynamics of a coarse-grained bead-spring polymer chain. The physics framework is:

- **Model:** Kremer-Grest bead-spring chain (N=30 beads primary, N=50/100/200 for OOD/scaling)
- **Dynamics:** Overdamped Langevin (Brownian dynamics) — the Rouse model with stochastic noise
- **Potentials:** FENE bonds (k=30, R₀=1.5σ) + WCA excluded volume (ε=1.0, σ=1.0)
- **Integrator:** Euler-Maruyama with additive noise
- **Goal:** Train a GNN to predict one-step displacements, evaluate via autoregressive rollout

The current codebase implements the basic simulator, force calculations, graph construction, baseline GNN, and rollout evaluation. The analysis of de Gennes' textbook identified **7 concrete enhancements** that should be implemented. They are ordered by priority (do #1–3 first, they're the highest value for effort).

---

## Task 1: Rouse Mode Analysis for Simulator Validation
**Priority:** 🔴 HIGH | **When:** Month 3 (simulator validation) | **Effort:** Medium

### Physics Background
The Rouse model decomposes polymer chain motion into independent normal modes. Each mode p has a specific relaxation time:

```
τ_p = τ₁ / p²

where τ₁ = τ_R = ζN²a² / (3π²k_BT)    [the Rouse time]
```

The Rouse mode coordinates are:

```
X_p(t) = (1/N) Σ_{n=1}^{N} r_n(t) · cos(πp(n − 1/2) / N)
```

for mode p = 0, 1, 2, ..., N−1.
- p=0 is the center of mass
- p=1 is the slowest internal mode (end-to-end breathing)
- p=2, 3, ... are faster, shorter-wavelength modes

The autocorrelation function of each mode decays exponentially:

```
⟨X_p(t) · X_p(0)⟩ = ⟨X_p²⟩ · exp(−t/τ_p)

where ⟨X_p²⟩ = Na² / (2π²p²)    for p ≥ 1 (ideal chain)
              = k_BT / (k_p)       where k_p = 12k_BT sin²(πp/2N) / a²
```

### What to Implement

**File:** Create `src/evaluation/rouse_modes.py`

```python
import numpy as np
from typing import Dict, List, Tuple

def compute_rouse_modes(
    trajectory: np.ndarray,  # shape (n_frames, N, dim)
    n_modes: int = 10,       # number of modes to compute (p=1..n_modes)
) -> np.ndarray:
    """
    Compute Rouse mode coordinates X_p(t) for a trajectory.
    
    X_p(t) = (1/N) Σ_n r_n(t) · cos(πp(n - 0.5) / N)
    
    Returns: shape (n_frames, n_modes, dim)
    """
    n_frames, N, dim = trajectory.shape
    modes = np.zeros((n_frames, n_modes, dim))
    
    for p in range(1, n_modes + 1):
        # Cosine basis function for mode p
        n_indices = np.arange(N) + 0.5  # n = 0.5, 1.5, ..., N-0.5
        cos_basis = np.cos(np.pi * p * n_indices / N)  # shape (N,)
        
        # Project positions onto mode p
        # modes[:, p-1, :] = (1/N) * Σ_n r_n(t) * cos(πp(n-0.5)/N)
        for d_idx in range(dim):
            modes[:, p-1, d_idx] = (1.0 / N) * np.dot(
                trajectory[:, :, d_idx], cos_basis
            )
    
    return modes


def compute_mode_autocorrelation(
    modes: np.ndarray,  # shape (n_frames, n_modes, dim)
    max_lag: int = None,
) -> np.ndarray:
    """
    Compute autocorrelation ⟨X_p(t) · X_p(0)⟩ for each mode.
    
    Returns: shape (max_lag, n_modes)
    """
    n_frames, n_modes, dim = modes.shape
    if max_lag is None:
        max_lag = n_frames // 4  # Use first quarter to avoid noise
    
    autocorr = np.zeros((max_lag, n_modes))
    
    for p in range(n_modes):
        mode_p = modes[:, p, :]  # shape (n_frames, dim)
        # Dot product across dimensions
        for lag in range(max_lag):
            n_samples = n_frames - lag
            # ⟨X_p(t+lag) · X_p(t)⟩ averaged over t and dimensions
            correlations = np.sum(
                mode_p[:n_samples] * mode_p[lag:lag+n_samples], axis=1
            )
            autocorr[lag, p] = np.mean(correlations)
    
    return autocorr


def fit_relaxation_times(
    autocorr: np.ndarray,  # shape (max_lag, n_modes)
    dt_save: float,         # time between saved frames (dt * save_every)
) -> Dict[str, np.ndarray]:
    """
    Fit exponential decay to each mode's autocorrelation to extract τ_p.
    
    Fit: C_p(t) = C_p(0) · exp(-t/τ_p)
    → log(C_p(t)/C_p(0)) = -t/τ_p
    → Linear fit in semi-log gives τ_p
    
    Returns dict with:
        'tau_p': array of relaxation times (shape n_modes)
        'tau_p_over_tau_1': normalized by τ₁ (should be ~1/p²)
        'rouse_scaling_valid': bool, whether τ_p ~ 1/p² holds
    """
    max_lag, n_modes = autocorr.shape
    tau_p = np.zeros(n_modes)
    
    for p in range(n_modes):
        c = autocorr[:, p]
        c_norm = c / c[0]  # Normalize by C(0)
        
        # Only fit where correlation is still positive and significant
        valid = c_norm > 0.05
        if np.sum(valid) < 5:
            tau_p[p] = np.nan
            continue
        
        t = np.arange(max_lag)[valid] * dt_save
        log_c = np.log(c_norm[valid])
        
        # Linear fit: log(C) = -t/τ + const
        # slope = -1/τ
        coeffs = np.polyfit(t, log_c, 1)
        if coeffs[0] < 0:
            tau_p[p] = -1.0 / coeffs[0]
        else:
            tau_p[p] = np.nan
    
    tau_1 = tau_p[0]
    tau_normalized = tau_p / tau_1 if tau_1 > 0 else tau_p
    
    # Check Rouse scaling: τ_p/τ_1 should ≈ 1/p²
    p_values = np.arange(1, n_modes + 1)
    expected = 1.0 / p_values**2
    
    # Relative error for first 5 modes
    valid_modes = ~np.isnan(tau_normalized[:5])
    if np.sum(valid_modes) >= 3:
        rel_error = np.abs(tau_normalized[:5][valid_modes] - expected[:5][valid_modes])
        rouse_valid = np.mean(rel_error) < 0.3  # 30% tolerance
    else:
        rouse_valid = False
    
    return {
        'tau_p': tau_p,
        'tau_p_over_tau_1': tau_normalized,
        'expected_rouse': expected,
        'rouse_scaling_valid': rouse_valid,
        'tau_rouse': tau_1,
    }
```

### Validation Criteria
- τ_p / τ₁ should approximately follow 1/p² for the first 5–8 modes
- For standard KG parameters with N=30: τ_R ≈ N²·dt / (3π²) in reduced units (rough estimate ~30–100 τ depending on γ)
- If this fails, the simulator has a dynamics bug that static checks (R_g, bond lengths) wouldn't catch

### Where This Fits in the Plan
- **Month 3 (§9):** Add Rouse mode analysis to `reports/drafts/simulator_validation.md`
- **Month 8+ (§9):** Use as GNN rollout evaluation metric (see Task 7 below)

---

## Task 2: MSD Subdiffusion Functions (g₁, g₂, g₃)
**Priority:** 🔴 HIGH | **When:** Month 3 (simulator validation) | **Effort:** Low

### Physics Background
De Gennes and the Rouse model predict distinct mean-squared displacement behaviors:

```
g₁(t) = ⟨(r_n(t) − r_n(0))²⟩          [monomer MSD]
       ~ t^{1/2}   for t ≪ τ_R          [subdiffusion!]
       ~ t¹         for t ≫ τ_R          [normal diffusion]

g₂(t) = ⟨((r_n(t) - R_cm(t)) − (r_n(0) - R_cm(0)))²⟩  [monomer MSD rel. to COM]
       ~ t^{1/2}   for t ≪ τ_R
       → const      for t ≫ τ_R          [saturates to ~R_g²]

g₃(t) = ⟨(R_cm(t) − R_cm(0))²⟩        [center-of-mass MSD]
       = 6 D_cm t = 6(k_BT / Nγ)t       [always linear, D ~ 1/N]
```

The t^{1/2} subdiffusion in g₁ is the **smoking gun** of Rouse dynamics.

### What to Implement

**File:** Create `src/evaluation/msd.py`

```python
import numpy as np
from typing import Dict, Tuple

def compute_msd_functions(
    trajectory: np.ndarray,  # shape (n_frames, N, dim)
    dt_save: float,           # time between saved frames
    max_lag_fraction: float = 0.25,
) -> Dict[str, np.ndarray]:
    """
    Compute g₁(t), g₂(t), g₃(t) MSD functions from a trajectory.
    
    Physics (de Gennes Ch. VI / Rouse model):
        g₁ = ⟨(r_n(t) − r_n(0))²⟩  (avg over n and time origins)
        g₂ = ⟨((r_n - R_cm)(t) − (r_n - R_cm)(0))²⟩
        g₃ = ⟨(R_cm(t) − R_cm(0))²⟩
    
    Returns dict with:
        't': time values
        'g1': monomer MSD (shape max_lag)
        'g2': monomer MSD rel. COM (shape max_lag)
        'g3': COM MSD (shape max_lag)
        'g1_exponent': fitted exponent α in g₁ ~ t^α (should be ~0.5 for t < τ_R)
        'g3_slope': fitted slope of g₃ vs t (= 6 D_cm)
        'D_cm': center-of-mass diffusion coefficient
    """
    n_frames, N, dim = trajectory.shape
    max_lag = int(n_frames * max_lag_fraction)
    
    # Center-of-mass trajectory
    r_cm = np.mean(trajectory, axis=1)  # shape (n_frames, dim)
    
    # Positions relative to COM
    r_rel = trajectory - r_cm[:, np.newaxis, :]  # shape (n_frames, N, dim)
    
    t = np.arange(1, max_lag + 1) * dt_save
    g1 = np.zeros(max_lag)
    g2 = np.zeros(max_lag)
    g3 = np.zeros(max_lag)
    
    for lag in range(1, max_lag + 1):
        n_origins = n_frames - lag
        
        # g₁: monomer MSD
        dr = trajectory[lag:] - trajectory[:n_origins]  # (n_origins, N, dim)
        g1[lag-1] = np.mean(np.sum(dr**2, axis=2))  # avg over n and origins
        
        # g₂: monomer MSD relative to COM
        dr_rel = r_rel[lag:] - r_rel[:n_origins]
        g2[lag-1] = np.mean(np.sum(dr_rel**2, axis=2))
        
        # g₃: COM MSD
        dr_cm = r_cm[lag:] - r_cm[:n_origins]
        g3[lag-1] = np.mean(np.sum(dr_cm**2, axis=1))
    
    # Fit g₁ exponent in the subdiffusive regime (first 1/4 of data)
    fit_range = max(5, max_lag // 4)
    log_t = np.log(t[:fit_range])
    log_g1 = np.log(g1[:fit_range] + 1e-30)
    g1_coeffs = np.polyfit(log_t, log_g1, 1)
    g1_exponent = g1_coeffs[0]  # Should be ~0.5 for Rouse
    
    # Fit g₃ slope (should be linear: g₃ = 6 D_cm t)
    g3_coeffs = np.polyfit(t, g3, 1)
    D_cm = g3_coeffs[0] / 6.0
    
    return {
        't': t,
        'g1': g1,
        'g2': g2,
        'g3': g3,
        'g1_exponent': g1_exponent,
        'g3_slope': g3_coeffs[0],
        'D_cm': D_cm,
    }
```

### Validation Criteria
- **g₁ exponent ≈ 0.5** for t < τ_R (subdiffusive regime) — this is the Rouse signature
- **g₃ linear in t** (center-of-mass diffuses normally)
- **D_cm ≈ k_BT/(Nγ)** — for N=30, γ=1.0, k_BT=1.0: D_cm ≈ 1/30 ≈ 0.033
- **g₂ saturates** to approximately 2R_g² at long times

### Integration Points
- Add to simulator validation (Month 3): run after pilot trajectories
- Add to GNN rollout evaluation (Month 8): compare g₁ exponent from ground truth vs. GNN rollout — if the GNN rollout gives g₁ ~ t^α with α ≠ 0.5, the model is not learning correct subdiffusive dynamics

---

## Task 3: Finite-Size Effective Exponent for Scaling Study
**Priority:** 🟡 MEDIUM | **When:** Month 10 (scaling study) | **Effort:** Low

### Physics Background
De Gennes' scaling law R_g ~ N^ν with ν ≈ 0.588 is an *asymptotic* result (N → ∞). For finite chains (N = 30, 50, 100, 200 — our actual simulation sizes), the *effective* exponent measured from a log-log fit can deviate from 0.588. This is expected, not a bug.

The standard approach (Kremer & Grest 1990, and see Rubinstein & Colby Ch. 3) is:

```
ν_eff = d(log R_g) / d(log N)

measured as the slope of log(R_g) vs log(N) from simulation data
```

For the KG model:
- N=30–50: ν_eff ≈ 0.55–0.57 (slightly below asymptotic due to FENE bond stiffness)
- N=100–200: ν_eff ≈ 0.57–0.585 (approaching asymptotic)
- N → ∞: ν_eff → 0.588

### What to Implement

**File:** Add to `src/evaluation/scaling.py` (or a new file)

```python
import numpy as np
from typing import Dict, List

def compute_effective_exponent(
    chain_lengths: List[int],    # e.g., [30, 50, 100, 200]
    rg_values: List[float],      # mean R_g for each chain length
    rg_errors: List[float] = None,  # stderr of R_g for each
) -> Dict[str, float]:
    """
    Fit R_g = A · N^ν_eff from simulation data.
    
    Also computes local (pairwise) exponents between consecutive N values
    to show the N-dependence of ν_eff.
    
    Returns:
        'nu_eff': global fitted exponent
        'nu_eff_err': fitting error
        'local_nu': list of pairwise exponents between consecutive N
        'ideal_nu': 0.5 (ideal chain reference)
        'saw_nu': 0.588 (SAW reference)
        'interpretation': string explaining result
    """
    log_N = np.log(np.array(chain_lengths, dtype=float))
    log_Rg = np.log(np.array(rg_values, dtype=float))
    
    # Global fit
    coeffs, cov = np.polyfit(log_N, log_Rg, 1, cov=True)
    nu_eff = coeffs[0]
    nu_err = np.sqrt(cov[0, 0])
    
    # Local (pairwise) exponents
    local_nu = []
    for i in range(len(chain_lengths) - 1):
        dlog_rg = log_Rg[i+1] - log_Rg[i]
        dlog_n = log_N[i+1] - log_N[i]
        local_nu.append(dlog_rg / dlog_n)
    
    # Interpretation
    if nu_eff < 0.52:
        interp = "Close to ideal chain (ν=0.5). Excluded volume effects may be too weak or chain too short."
    elif nu_eff < 0.57:
        interp = f"ν_eff = {nu_eff:.3f} ± {nu_err:.3f}. Between ideal (0.5) and SAW (0.588). Expected for finite-N KG chains — consistent with de Gennes/Kremer-Grest."
    elif nu_eff < 0.60:
        interp = f"ν_eff = {nu_eff:.3f} ± {nu_err:.3f}. Close to SAW limit (0.588). Good agreement with de Gennes' prediction."
    else:
        interp = f"ν_eff = {nu_eff:.3f} ± {nu_err:.3f}. Above SAW limit — possible systematic error or non-equilibrium effect."
    
    return {
        'nu_eff': nu_eff,
        'nu_eff_err': nu_err,
        'local_nu': local_nu,
        'ideal_nu': 0.5,
        'saw_nu': 0.588,
        'interpretation': interp,
    }
```

### Why This Matters
Without this, the Month 10 scaling study might naively fit ν = 0.55 from N=30–200 data and worry that "the physics is wrong." De Gennes' framework explains exactly why finite-N gives a lower effective exponent, and the local exponent should *increase toward 0.588* as N increases. The GNN scaling study should show the same trend.

---

## Task 4: Rouse-Informed Physics Loss Term (Optional, Month 7+)
**Priority:** 🟡 MEDIUM | **When:** Month 7–9 | **Effort:** Medium

### Physics Background
De Gennes' Rouse model predicts specific constraints on how displacement magnitudes should relate to chain position. Specifically:
- End beads (n=0, N-1) have more freedom than interior beads
- The displacement variance of bead n depends on its chain position via the Rouse mode amplitudes

Currently, the physics-informed loss in [losses.py](file:///Users/k.siddharthareddy/Documents/BASHI+OK/PINN/src/training/losses.py) only has bond-length and excluded-volume penalties. A Rouse-mode-inspired loss would penalize predictions that violate the mode structure.

### What to Implement

**File:** Add to `src/training/losses.py`

```python
def rouse_mode_consistency_loss(
    current_positions: torch.Tensor,     # (N, 3) current frame
    predicted_displacement: torch.Tensor,  # (N, 3) GNN prediction
    n_modes: int = 5,
) -> torch.Tensor:
    """
    Penalize predictions that violate Rouse mode amplitude hierarchy.
    
    Physics: The variance of mode p should scale as 1/p².
    If the predicted displacement projects strongly onto high-p modes
    (fast oscillations) without proportional low-p contribution, it's
    likely unphysical.
    
    This is a SOFT constraint — it doesn't force exact Rouse behavior
    but penalizes clearly non-Rouse displacement patterns.
    """
    predicted_positions = current_positions + predicted_displacement
    N = predicted_positions.shape[0]
    
    # Compute Rouse mode amplitudes of the predicted displacement
    mode_amplitudes = torch.zeros(n_modes)
    for p in range(1, n_modes + 1):
        n_idx = torch.arange(N, dtype=torch.float32, device=predicted_displacement.device) + 0.5
        cos_basis = torch.cos(torch.pi * p * n_idx / N)
        
        # Project displacement onto mode p
        for d in range(3):
            projection = torch.dot(predicted_displacement[:, d], cos_basis) / N
            mode_amplitudes[p-1] += projection**2
    
    # Expected Rouse scaling: amplitude_p² ∝ 1/p²
    # Penalize deviations from this hierarchy
    p_values = torch.arange(1, n_modes + 1, dtype=torch.float32, device=predicted_displacement.device)
    expected_ratio = (mode_amplitudes[0] / p_values**2)  # expected amplitudes based on p=1
    
    # Only penalize when higher modes have disproportionately large amplitudes
    excess = torch.relu(mode_amplitudes[1:] - 2.0 * expected_ratio[1:])
    
    return torch.mean(excess)
```

> [!NOTE]
> This is an **experimental** loss term — it should be an ablation row, not a default. Add it to the ablation matrix (§11) as a separate row to test whether Rouse-mode regularization helps rollout stability.

---

## Task 5: Fluctuation-Dissipation Temperature Check
**Priority:** 🟡 MEDIUM | **When:** Month 3 | **Effort:** Low

### Physics Background
De Gennes' dynamics framework rests on the fluctuation-dissipation theorem (FDT). For overdamped Brownian dynamics:

```
⟨ξ(t)·ξ(t')⟩ = 2γk_BT · δ(t−t') · I

This means: ⟨|Δr|²⟩_noise = 2k_BT·dt/γ · dim
```

The temperature can be *estimated* from displacement statistics:

```
T_est = γ · ⟨|Δr|²⟩_noise / (2 · dt · dim)
```

where Δr_noise is the noise contribution (total displacement minus the deterministic drift).

### What to Implement

**File:** Add to `src/evaluation/temperature_check.py`

```python
def estimate_temperature_from_displacements(
    trajectory: np.ndarray,  # (n_frames, N, dim)
    forces_trajectory: np.ndarray,  # (n_frames, N, dim) — forces at each frame
    dt: float,
    gamma: float,
    dim: int = 3,
) -> Dict[str, float]:
    """
    Estimate effective temperature from displacement statistics.
    
    Following execution plan §3 and de Gennes' FDT:
    Total displacement = drift + noise
    drift = (dt/γ) * F
    noise = displacement - drift
    
    T_est = γ · ⟨|noise|²⟩ / (2 · dt · dim)
    
    This should give T_est ≈ k_BT (target temperature).
    """
    n_frames = trajectory.shape[0]
    
    displacements = trajectory[1:] - trajectory[:-1]  # (n_frames-1, N, dim)
    drift = (dt / gamma) * forces_trajectory[:-1]      # (n_frames-1, N, dim)
    noise = displacements - drift
    
    # Mean squared noise per bead per frame
    noise_sq = np.mean(np.sum(noise**2, axis=2))  # avg over N and frames
    
    T_est = gamma * noise_sq / (2.0 * dt * dim)
    
    return {
        'T_estimated': T_est,
        'T_target': 1.0,  # k_BT = 1.0 in reduced units
        'relative_error': abs(T_est - 1.0) / 1.0,
        'passed': abs(T_est - 1.0) / 1.0 < 0.1,  # within 10%
    }
```

### Why This Matters
The execution plan §3 explicitly says: "estimate temperature from displacement/diffusion statistics, NOT from a raw kinetic-energy readout." This implements exactly that check. De Gennes' entire dynamics framework assumes the FDT holds — if this check fails, the integrator has a bug.

---

## Task 6: Report Text — Rouse/Zimm/Solvent Framing
**Priority:** 🟡 MEDIUM | **When:** Month 12 (report writing) | **Effort:** Zero (just text)

### Recommended Text for Report Section "Physics Background"

The coding agent should include this framing in the report draft:

> **Dynamics regime:** Our simulation employs overdamped Brownian dynamics (the Langevin equation in the high-friction limit), which corresponds to the **Rouse model** for polymer dynamics (de Gennes, 1979, Ch. VI; Rouse, 1953). In this regime, each bead experiences a friction coefficient γ against an implicit solvent, with no hydrodynamic interactions (HI) between beads. This is the "freely draining" limit.
>
> This choice is appropriate for two reasons: (1) our coarse-grained Kremer-Grest model uses implicit solvent, where HI is absent by construction, and (2) in polymer melts, hydrodynamic interactions are screened by surrounding chains (de Gennes, 1979, Ch. II), making the Rouse model the physically correct description even in the presence of solvent. The alternative **Zimm model**, which includes HI and predicts different scaling (τ ~ N^{3ν} vs. τ ~ N² for Rouse), applies to dilute solutions in explicit solvent and is outside our scope.
>
> **Solvent quality:** The WCA (Weeks-Chandler-Andersen) potential used for excluded-volume interactions is purely repulsive, placing our simulations in the **good-solvent** regime throughout. The chain is a self-avoiding walk (SAW) with R_g ~ N^ν, ν ≈ 0.588 (de Gennes, 1979, §I.3). We do not explore the theta-solvent (ν = 1/2) or poor-solvent (ν = 1/3) regimes, which would require adding an attractive interaction tail.

---

## Task 7: Rouse Mode Spectrum as GNN Rollout Evaluation Metric
**Priority:** 🔴 HIGH | **When:** Month 8+ | **Effort:** Medium

### Physics Background
The most scientifically informative way to evaluate whether a GNN has learned the correct polymer dynamics is to check whether its rollout trajectories reproduce the Rouse mode relaxation spectrum.

Standard metrics (R_g, bond lengths, one-step MSE) test *structural* properties. The Rouse mode spectrum tests the *dynamics across all length scales simultaneously*.

### What to Implement

**File:** Add to `src/evaluation/rollout.py` (extend `RolloutEvaluator`)

```python
def evaluate_rouse_mode_spectrum(
    ground_truth_trajectory: np.ndarray,  # (n_frames, N, 3)
    rollout_trajectory: np.ndarray,        # (n_frames_rollout, N, 3)
    dt_save: float,
    n_modes: int = 8,
) -> Dict[str, Any]:
    """
    Compare Rouse mode relaxation spectra between ground truth and GNN rollout.
    
    For each mode p:
    1. Compute autocorrelation C_p(t)
    2. Fit relaxation time τ_p
    3. Compare GT τ_p vs rollout τ_p
    4. Check whether τ_p/τ_1 ~ 1/p² holds for the rollout
    
    Returns:
        'gt_tau': ground truth relaxation times
        'rollout_tau': rollout relaxation times
        'tau_ratio': rollout/GT for each mode (should be ~1.0)
        'gt_rouse_valid': whether GT follows Rouse scaling
        'rollout_rouse_valid': whether rollout follows Rouse scaling
        'mode_fidelity': overall score (0-1) measuring how well rollout
                         reproduces the mode spectrum
    """
    from .rouse_modes import compute_rouse_modes, compute_mode_autocorrelation, fit_relaxation_times
    
    # Ground truth
    gt_modes = compute_rouse_modes(ground_truth_trajectory, n_modes)
    gt_autocorr = compute_mode_autocorrelation(gt_modes)
    gt_result = fit_relaxation_times(gt_autocorr, dt_save)
    
    # Rollout
    roll_modes = compute_rouse_modes(rollout_trajectory, n_modes)
    roll_autocorr = compute_mode_autocorrelation(roll_modes)
    roll_result = fit_relaxation_times(roll_autocorr, dt_save)
    
    # Compare
    valid = ~np.isnan(gt_result['tau_p']) & ~np.isnan(roll_result['tau_p'])
    if np.sum(valid) == 0:
        return {'mode_fidelity': 0.0, 'error': 'No valid modes to compare'}
    
    tau_ratio = roll_result['tau_p'][valid] / gt_result['tau_p'][valid]
    
    # Mode fidelity: fraction of modes where rollout τ is within 2x of GT
    fidelity = np.mean(np.abs(np.log(tau_ratio)) < np.log(2.0))
    
    return {
        'gt_tau': gt_result['tau_p'],
        'rollout_tau': roll_result['tau_p'],
        'tau_ratio': tau_ratio,
        'gt_rouse_valid': gt_result['rouse_scaling_valid'],
        'rollout_rouse_valid': roll_result['rouse_scaling_valid'],
        'mode_fidelity': float(fidelity),
    }
```

### Why This Is the Highest-Value Evaluation Metric
- A model that reproduces R_g but gets the Rouse spectrum wrong has learned static structure but NOT dynamics
- A model that gets the Rouse spectrum right has learned the correct multi-scale relaxation across ALL length scales
- This is exactly the kind of metric that distinguishes "the physics-informed model genuinely learned better dynamics" from "it just matches aggregate statistics by coincidence"
- **No published GNN-for-polymer-dynamics paper we're aware of uses Rouse mode analysis as an evaluation metric** — this would be a novel, scientifically rigorous contribution

---

## Summary of All Tasks

| # | Task | File | Priority | Month | Lines of Code |
|---|------|------|----------|-------|---------------|
| 1 | Rouse mode analysis | `src/evaluation/rouse_modes.py` [NEW] | 🔴 HIGH | 3 | ~100 |
| 2 | MSD functions (g₁, g₂, g₃) | `src/evaluation/msd.py` [NEW] | 🔴 HIGH | 3 | ~70 |
| 3 | Finite-size ν_eff | `src/evaluation/scaling.py` [NEW] | 🟡 MEDIUM | 10 | ~50 |
| 4 | Rouse mode loss term | `src/training/losses.py` [MODIFY] | 🟡 MEDIUM | 7–9 | ~40 |
| 5 | FDT temperature check | `src/evaluation/temperature_check.py` [NEW] | 🟡 MEDIUM | 3 | ~30 |
| 6 | Report text framing | Report draft | 🟡 MEDIUM | 12 | ~0 (text) |
| 7 | Rouse spectrum rollout eval | `src/evaluation/rollout.py` [MODIFY] | 🔴 HIGH | 8+ | ~60 |

**Total new code: ~350 lines.** All implementations are self-contained with minimal dependencies on existing code.

---

## Key Equations Reference Card

For quick reference, here are all the de Gennes equations that should be verifiable in our simulation:

| Equation | Source | Our Validation |
|----------|--------|---------------|
| R_g ~ N^ν, ν ≈ 0.588 | de Gennes §I.3 (Eq. I.39) | Month 10 scaling study |
| ⟨r²⟩ = Na² (ideal chain) | de Gennes Eq. I.4 | Baseline comparison |
| τ_R ~ N² | de Gennes §VI.1 (Eq. VI.8) | Task 1: Rouse mode analysis |
| τ_p ~ N²/p² | de Gennes Eq. VI.8 | Task 1: mode spectrum |
| g₁ ~ t^{1/2} (subdiffusion) | Rouse theory | Task 2: MSD functions |
| D_cm = k_BT/(Nγ) | de Gennes Eq. VI.18 | Task 2: g₃ slope check |
| ⟨\|ξ\|²⟩ = 2k_BTdt/γ (FDT) | de Gennes Ch. VI / §3 of plan | Task 5: temperature check |
| Bond length ≈ 0.965σ (FENE+WCA) | Kremer & Grest 1990 | Already in plan §13.1 |
