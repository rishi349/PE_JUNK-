# Fix Plan: All 7 Review Issues

## Fixes Overview

| # | Issue | Severity | Files Changed |
|---|-------|----------|---------------|
| 1 | Dead `graph_builder.py` module | Medium | Delete `src/data/graph_builder.py`, update `src/data/__init__.py` |
| 2 | Missing Z-score normalization | High | New `src/data/normalization.py`, modify `trainer.py`, `dataset.py` |
| 3 | Checksum bug (self-referential) | Medium | `src/data/trajectory.py`, `src/simulator/numpy_simulator.py` |
| 4 | Burn-in units wrong (~180x too short) | **Critical** | `configs/default.yaml`, docs/comments |
| 5 | Harmonic+WCA equilibrium claim wrong | Low | `configs/default.yaml` comment, `validate_simulator.py` |
| 6 | Trajectory count mismatch (500 vs 150) | Low | `configs/default.yaml` |
| 7 | Vectorize force loops (perf) | High | `src/physics/forces.py`, new tests |

---

## Fix Details

### Fix #6: Trajectory count mismatch
Update `configs/default.yaml` to use 150 (plan's number) instead of 500.

### Fix #3: Checksum bug
**Problem**: File is saved → checksum computed on saved file → checksum embedded in metadata → file re-saved. The final file's hash won't match the stored checksum.
**Solution**: Compute checksum on the *final* file contents. Save once with a placeholder, compute checksum, then store it in a *separate sidecar file* (`trajectory_XXXX.sha256`). This avoids the circular dependency entirely. `verify_checksum` reads from the sidecar.

### Fix #1: Remove dead `graph_builder.py`
Delete the file entirely. It's Month 2 scaffolding that was superseded by `graph_construction.py`. Update `__init__.py` accordingly.

### Fix #5: Harmonic+WCA equilibrium comment
Change the config comment and validation script's default `--expected-bond-length` from `1.0` to `~1.06` (the real emergent equilibrium under harmonic + WCA).

### Fix #4: Burn-in units (CRITICAL)
**Problem**: `n_burnin: 5000` steps at `dt=0.001` = 5τ. But a Rouse time for N=30 is ~900τ. Need ~2700τ minimum (3 Rouse times).
**Solution**: Change `n_burnin` to `2700000` steps (= 2700τ at dt=0.001). Also increase `T_steps` to `3700000` (2.7M burn-in + 1M production = 1000τ of usable data). Update the comment to show the unit conversion explicitly.

### Fix #7: Vectorize force loops
Replace the pure-Python `for i / for j` loops in `compute_bonded_forces` and `compute_nonbonded_forces` with vectorized NumPy operations (pairwise distance matrices, broadcasting). Keep the pairwise functions as reference implementations for testing.

### Fix #2: Add Z-score normalization
Create `src/data/normalization.py` with a `DisplacementNormalizer` class that:
- Computes mean/std of displacement targets across the training set
- Normalizes targets before training (`(y - mean) / std`)
- Denormalizes predictions after inference (`pred * std + mean`)
- Saves/loads statistics for reproducibility
Integrate into `Trainer` and `RolloutEvaluator`.

---

## Commit Schedule

| Date | Day | Commits |
|------|-----|---------|
| Aug 8 (Fri) | Evening | Fix #6: trajectory count config |
| Aug 9 (Sat) | Morning/Afternoon/Evening | Fix #3: checksum, Fix #1: dead code, Fix #5: equilibrium |
| Aug 10 (Sun) | Morning/Afternoon/Evening | Fix #4: burn-in, Fix #7 part 1+2: vectorize forces |
| Aug 11 (Mon) | Morning/Afternoon | Fix #4: tests, Fix #2 part 1: normalization module |
| Aug 12 (Tue) | — | *No commits* |
| Aug 13 (Wed) | Midday | Fix #2 part 2: integrate normalization |
| Aug 14 (Thu) | — | *No commits* |
| Aug 15 (Fri) | Afternoon | Final cleanup, run tests |

> [!IMPORTANT]
> No git push until you explicitly tell me to. All commits are local only.
