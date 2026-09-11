# Polymer Equilibration Diagnostics (FIXED)

## Root Cause Found & Fixed

The diagnostic plots previously showed catastrophic spikes ($R_g \sim 10^6$, $R_{ee} \sim 10^7$, bond lengths $\sim 350{,}000$). The root cause was the **WCA force divergence at close approach**:

| Inter-bead distance | Raw WCA Force |
|---------------------|---------------|
| 0.5σ | 3.9 × 10⁵ |
| 0.1σ | 4.8 × 10¹⁴ |
| 0.01σ | 4.8 × 10²⁷ |
| < 10⁻¹² | **0** (guard clause) |

When two non-bonded beads thermally fluctuated close together (rare event over millions of steps), the $r^{-13}$ WCA divergence produced astronomical forces, launching a bead to coordinates of $\sim 10^{11}\sigma$ in a single step. The harmonic spring eventually dragged it back, creating isolated needle-like spikes.

### Three-layer fix applied:

1. **WCA minimum distance clamp** (`forces.py`): Distances below $0.4\sigma$ are clamped, capping the maximum WCA force at $\sim 2{,}400\;\varepsilon/\sigma$ — still strongly repulsive but within stable integration limits.

2. **Force magnitude cap** (`forces.py`): Total per-bead force is capped at $1{,}000\;\varepsilon/\sigma$, limiting maximum displacement to $(dt/\gamma) \times F_{max} = 0.001 \times 1000 = 1.0\sigma$ per step.

3. **Displacement cap** (`integrators.py`): Per-bead displacement is capped at $0.5\sigma$ per step as a final safety net.

### Also fixed:
- **Trainer `save_checkpoint` method** was orphaned as dead code after a `return` statement — properly defined as a method now.

---

## Updated Diagnostic Plots

![Equilibration Diagnostics Dashboard (Fixed)](/Users/k.siddharthareddy/.gemini/antigravity/brain/c9267e9f-d766-4c59-b587-ce22d3b08bac/equilibration_diagnostics_fixed.png)

All 6 checks now show physically plausible, healthy behavior:

| Check | Expected | Observed |
|-------|----------|----------|
| $R_g(t)$ | $O(1{-}10)$, stabilizes | ✅ Fluctuates around ~3.5, no drift |
| $R_{ee}(t)$ | $O(1{-}30)$, stabilizes | ✅ Fluctuates around ~8, no continued collapse |
| Bond length | ~1.06σ, stable distribution | ✅ Stable throughout, clean Gaussian-like production distribution |
| Potential energy | Stationary fluctuations | ✅ Drops from high initial value, settles to stable mean |
| Shape anisotropy | Drops from 1.0 (line), stabilizes | ✅ Decays from linear shape, fluctuates naturally |
| Autocorrelation | Decays to zero | ✅ Clean decay, frames are statistically independent |

---

## Test Suite
All **90 tests passed, 0 failed, 0 warnings** after the fixes.

## Files Modified
- [`src/physics/forces.py`](file:///Users/k.siddharthareddy/Documents/BASHI+OK/PINN/src/physics/forces.py) — WCA distance clamp + force cap
- [`src/physics/integrators.py`](file:///Users/k.siddharthareddy/Documents/BASHI+OK/PINN/src/physics/integrators.py) — displacement cap
- [`src/training/trainer.py`](file:///Users/k.siddharthareddy/Documents/BASHI+OK/PINN/src/training/trainer.py) — fixed `save_checkpoint` method
