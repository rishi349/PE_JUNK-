# De Gennes — "Scaling Concepts in Polymer Physics" (1979)
## Concepts Extracted for the Physics-Informed GNN Polymer Project

> **Source:** Pierre-Gilles de Gennes, *Scaling Concepts in Polymer Physics*, Cornell University Press, 1979. 325 pages, 11 chapters.
>
> **Cross-referenced against:**
> - [polymer_gnn_1_year_execution_plan.md](file:///Users/k.siddharthareddy/Documents/BASHI+OK/polymer_gnn_1_year_execution_plan.md)
> - Codebase in [PINN/src/](file:///Users/k.siddharthareddy/Documents/BASHI+OK/PINN/src)
>
> **How to read this document:** Each concept from de Gennes is tagged:
> - **✅ ADOPTED** — Already in our plan/code, with location reference
> - **🟡 PARTIALLY ADOPTED** — Concept is touched but not fully exploited
> - **🔴 NOT YET ADOPTED** — Relevant concept we should consider adding
> - **⬜ IRRELEVANT** — Not applicable to our project scope, with reason

---

## Book Structure Overview

| Part | Chapters | Topic | Relevance to Our Project |
|------|----------|-------|--------------------------|
| **Intro** | — | Long Flexible Chains | ⭐⭐⭐ Core framing |
| **Part A: Static Conformations** | I–V | Single chain, melts, solutions, phase separation, gels | ⭐⭐⭐ Ch I critical, rest contextual |
| **Part B: Dynamics** | VI–VIII | Single-chain dynamics, many-chain dynamics, entanglement/reptation | ⭐⭐⭐⭐ Most directly relevant |
| **Part C: Calculation Methods** | IX–XI | Self-consistent fields, polymer–critical phenomena mapping, renormalization group | ⭐ Advanced theory, not directly coded |

---

## Introduction: Long Flexible Chains (pp. 19–28)

### Concept 1: The Universality of Polymer Chain Statistics
**De Gennes' Key Insight:** Despite enormous chemical diversity (polyethylene, polystyrene, PMMA, PDMS, etc.), the *statistical* properties of long flexible chains — end-to-end distance, R_g, MSD — are governed by universal scaling laws that depend only on chain length N, dimensionality d, and solvent quality, NOT on chemical details.

**Status: ✅ ADOPTED**
- **Where:** The entire project premise. Our execution plan §1.4 explicitly states: "the standard shape statistics (radius of gyration, end-to-end distance) you'll use for validation." The project uses a *coarse-grained* (Kremer-Grest) bead-spring chain, which is the computational embodiment of de Gennes' universality claim.
- **Why it matters for us:** This universality is *why* a GNN trained on one chemical "species" (KG beads) can be expected to learn transferable physics. Our OOD tests (§9, Month 9) probe whether the GNN captures this universality across chain lengths.

### Concept 2: The Gaussian Chain / Ideal Chain (R ~ N^{1/2})
**De Gennes, Eq. (I.4):** ⟨r²⟩ = Na² (= R₀²), so R₀ ~ N^{1/2} · a.

An ideal chain is a random walk where each step is independent. The end-to-end distance distribution is Gaussian (Eq. I.6). This is the *baseline expectation* against which excluded-volume effects are measured.

**Status: ✅ ADOPTED**
- **Where:** Plan §1.4, §4 parameter table, §13.1 validation. We initialize chains as random walks ([numpy_simulator.py](file:///Users/k.siddharthareddy/Documents/BASHI+OK/PINN/src/simulator/numpy_simulator.py#L136-L140) `random_walk` method).
- **Comparison with other works:** Rubinstein & Colby Ch. 2 gives the same result but with more pedagogical detail. Doi & Edwards (1986) derive it from the partition function. De Gennes is unique in immediately connecting this to the *scaling* framework rather than treating it as a standalone statistical result.

---

## Chapter I: A Single Chain (pp. 29–53)

### Concept 3: Self-Avoiding Walk (SAW) and the Flory Exponent ν
**De Gennes, §I.2–I.3:** A *real* chain in a good solvent cannot overlap with itself (excluded volume). The chain swells beyond the ideal R₀. Flory's mean-field calculation (balancing elastic entropy against excluded-volume repulsion) gives:

> **R_F ~ N^ν**, where **ν = 3/(d+2)**
>
> In 3D: ν = 3/5 = 0.6 (Flory), refined to **ν ≈ 0.588** by renormalization group (Eq. I.39, p. 45)

This is arguably the single most important equation in polymer physics. The exponent ν controls how chain size scales with chain length.

**Status: ✅ ADOPTED**
- **Where:** Plan §2 (Month 2): "expected R_g scaling (R_g ~ N^ν with ν ≈ 0.588 for a good solvent / self-avoiding walk)." Plan §13.1: used as validation target. Month 10 scaling study explicitly checks this.
- **In code:** [numpy_simulator.py](file:///Users/k.siddharthareddy/Documents/BASHI+OK/PINN/src/simulator/numpy_simulator.py#L173-L177) computes R_g, and the plan cross-checks R_g(N) scaling against ν ≈ 0.588 in Months 3 and 10.

### Concept 4: The Excluded Volume Parameter and When It Matters
**De Gennes, §I.3.2 (p. 45):** The dimensionless expansion parameter ζ = (v/a^d) · N^{2−d/2} determines whether excluded volume matters. When ζ ≪ 1, the chain is effectively ideal. When ζ ≫ 1, it's strongly self-avoiding.

In d=3: ζ ~ v · N^{1/2} / a³. For a chain of N=30 beads with standard KG parameters (v ~ σ³, a ~ σ), ζ is moderate — excluded volume is *present but not overwhelming*.

**Status: 🟡 PARTIALLY ADOPTED**
- **Where:** Plan §4 uses WCA (purely repulsive LJ) for excluded volume, which is correct. The WCA implementation in [forces.py](file:///Users/k.siddharthareddy/Documents/BASHI+OK/PINN/src/physics/forces.py#L205-L272) is the computational equivalent.
- **Gap:** We don't explicitly compute or track ζ as a diagnostic. For N=30, ζ is moderate, meaning our chains are in a crossover regime between ideal and fully swollen — this is actually important to acknowledge when interpreting the scaling study (Month 10), because R_g ~ N^ν with ν ≈ 0.588 is the *asymptotic* limit (N→∞). For short chains (N=30–50), finite-size corrections to the scaling exponent are expected, and fitting a power law naively could give ν slightly different from 0.588 without any physics being wrong.
- **Recommendation:** Add a note to the scaling study analysis that measures the *effective* exponent from a log-log fit and compares it to both 0.5 (ideal) and 0.588 (SAW), expecting something in between for short chains.

### Concept 5: Pair Correlations Inside a Swollen Coil
**De Gennes, §I.2.3 (p. 42):** The pair correlation function g(r) inside a single swollen chain decays as g(r) ~ r^{-(d−2+η)}, where η is a critical exponent related to the polymer-magnet correspondence. In 3D, η ≈ 0.03 (small but nonzero).

**Status: ⬜ IRRELEVANT**
- **Why:** This is a subtle structural correlation that matters for scattering experiments (SANS, light scattering). Our project doesn't measure or predict pair correlation functions — we predict displacements. The GNN implicitly captures local pairwise structure through its message-passing on edges, but we don't need to explicitly verify the η exponent.

### Concept 6: Constrained Chains — Chain Under Traction and Chain in a Tube
**De Gennes, §I.4 (pp. 46–53):**
- **Under traction (§I.4.1):** When an external force f stretches the chain, the end-to-end distance grows nonlinearly. For strong stretching: r ~ (fR_F/T)^{δ−1} · R_F, where δ = (1−ν)^{−1} ≈ 5/2 in 3D.
- **In a tube (§I.4.2):** The "blob" picture first appears here. A chain confined in a tube of diameter D forms a sequence of independent blobs, each containing g ~ (D/a)^{1/ν} monomers. The chain length along the tube: R_∥ ~ N · a · (a/D)^{1/ν − 1}.

**Status: ⬜ IRRELEVANT for current scope / 🔴 POTENTIALLY RELEVANT for future work**
- **Why currently irrelevant:** Our chains are free in solution (no external forces, no confinement). The plan uses periodic boundary conditions with a large box (no confinement effects).
- **Why potentially interesting for future work:** If the project ever extends to confined polymers (e.g., polymer in a nano-channel), the blob model for confinement gives exact scaling predictions the GNN should recover. This could be a powerful OOD test: train on free chains, test on confined chains. But this is beyond the 1-year scope.

---

## Chapter II: Polymer Melts (pp. 54–68)

### Concept 7: Screening of Excluded Volume in Dense Systems (Flory's Ideality Theorem)
**De Gennes, §II.1 (pp. 54–61):** In a dense melt, excluded-volume interactions are *screened* — each chain behaves as an ideal chain (ν = 1/2) even though individual monomer-monomer repulsions still exist. This is because the chain is surrounded by many *other* chains whose monomers fill space uniformly, canceling the net excluded-volume effect.

> **Key result:** In a melt, R_g ~ N^{1/2} (ideal scaling), NOT N^{0.588}

This is one of de Gennes' most celebrated results, building on Flory's earlier argument but placing it on a rigorous scaling foundation.

**Status: ⬜ IRRELEVANT for current scope**
- **Why:** Our simulation is a *single chain* in implicit solvent (no other chains present). The plan explicitly states (§16): "we study a single coarse-grained polymer chain." Dense-melt screening is a many-chain phenomenon.
- **If scope ever expanded:** For multi-chain systems, the GNN would need to learn this screening behavior — chains in a melt should have ν=1/2, not ν=0.588. This would be a fascinating test of physics-informed learning but requires multi-chain simulation infrastructure we don't have.

### Concept 8: The Correlation Hole
**De Gennes, §II.2.2 (pp. 62–64):** In a melt, a labeled chain's monomers displace other chains' monomers from their vicinity, creating a "correlation hole" in the inter-chain pair correlation function. The hole extends to ~R_g.

**Status: ⬜ IRRELEVANT**
- **Why:** Single-chain simulation — no inter-chain correlations exist.

---

## Chapter III: Polymer Solutions in Good Solvents (pp. 69–97)

### Concept 9: The Overlap Concentration c* and the Blob Model
**De Gennes, §III.2.1 (p. 78):** The overlap concentration c* marks the crossover from dilute to semi-dilute behavior:

> c* ~ N / R_F^3 ~ N^{1−3ν} ~ N^{−4/5} (in 3D with ν = 3/5)

Above c*, chains overlap and the solution can be described as a network of "blobs" of size ξ (correlation length / screening length):

> **ξ ~ c^{−ν/(3ν−1)} ~ c^{−3/4}** (good solvent, 3D)

Inside each blob: SAW statistics (ν ≈ 0.588). Between blobs: ideal random walk (ν = 1/2). This is the birth of the "blob" concept that pervades modern polymer physics.

**Status: ⬜ IRRELEVANT for current scope / 🔴 VALUABLE CONCEPTUAL CONTEXT**
- **Why currently irrelevant:** Single dilute chain, no concentration effects.
- **Why valuable context:** The blob model is *conceptually* how our GNN's multi-scale message passing works — local messages capture short-range correlations (like intra-blob physics), while deeper layers propagate information across larger scales (like inter-blob connectivity). This analogy could strengthen the "Related Work / Background" section of the final report (§18).

### Concept 10: Correlation Length and Screening in Semi-Dilute Solutions
**De Gennes, §III.2.4–III.2.7:** In semi-dilute solutions, excluded-volume interactions are screened beyond the correlation length ξ. The chain conformations on scales > ξ follow ideal statistics.

**Status: ⬜ IRRELEVANT** — Same reason as above.

---

## Chapter IV: Incompatibility and Segregation (pp. 98–127)

### Concept 11: Phase Separation and Theta Solvents
**De Gennes, §IV.3 (pp. 113–121):** Near the theta temperature (T = Θ), the excluded volume parameter v → 0, and chains become ideal (ν = 1/2). Below Θ (poor solvent), chains collapse (ν = 1/3 in 3D).

**Status: 🟡 PARTIALLY ADOPTED**
- **Where in plan:** The plan's OOD grid (§9, Month 9) tests temperature variation but doesn't explicitly consider the theta-to-good-solvent crossover.
- **Relevance:** Our WCA potential is *always repulsive* (good solvent regime). There is no attractive tail, so we never enter the theta or poor-solvent regime. This is by design (standard KG model), not an omission.
- **Recommendation:** Worth mentioning in the report's "Limitations" section (§16): "Our simulations use the WCA (purely repulsive) potential, which corresponds to good-solvent conditions throughout. The model is not tested in theta or poor-solvent regimes where chain collapse would occur."

---

## Chapter V: Polymer Gels (pp. 128–163)

### Concept 12: Gelation and Percolation
**De Gennes, §V.2:** Gel formation is mapped onto percolation theory. Cross-linked polymer networks exhibit critical behavior near the gel point, with universal exponents.

**Status: ⬜ IRRELEVANT**
- **Why:** We simulate a *linear* single chain, not a cross-linked network. Gelation requires multi-chain systems with cross-linking chemistry.

---

## Chapter VI: Dynamics of a Single Chain (pp. 165–204) — ⭐ MOST RELEVANT CHAPTER

This is the chapter most directly relevant to our project. Almost every equation here connects to something we're already doing or should be doing.

### Concept 13: The Rouse Model — Bead-Spring Chain Dynamics
**De Gennes, §VI.1 (pp. 165–172):** The Rouse model is the foundational description of a single polymer chain's dynamics:

> **Core equation (Eq. VI.5):** ∂r_n/∂t = (3Tμ/a²) · ∂²r/∂n²
>
> - Chain = beads connected by harmonic springs
> - Each bead has friction coefficient ζ = 1/μ against the solvent
> - **Phantom chain** assumption: chain can cross itself
> - **Locality of response**: bead n only feels forces from neighbors n±1
>
> **Normal modes (Eq. VI.7):** r_{np}(t) = cos(πpn/N) · exp(−t/τ_p) · α_p
>
> **Rouse relaxation time (Eq. VI.8):** τ_p ∝ (N/p)², so **τ_R = τ₁ ∝ N²**

**Status: ✅ ADOPTED — but deeper exploitation possible**
- **Where adopted:**
  - Plan §1.2–1.3: Brownian dynamics / overdamped Langevin is the Rouse model with stochastic noise
  - Plan §4: burn-in ≥ "a few Rouse times (~N² in reduced units)" — this directly uses τ_R ~ N²
  - [integrators.py](file:///Users/k.siddharthareddy/Documents/BASHI+OK/PINN/src/physics/integrators.py#L30-L90): Euler-Maruyama overdamped step implements the stochastic version of Rouse dynamics
  - [forces.py](file:///Users/k.siddharthareddy/Documents/BASHI+OK/PINN/src/physics/forces.py#L39-L80): harmonic bond force is exactly the Rouse spring

- **🔴 NOT YET EXPLOITED — Rouse Mode Analysis:**
  - De Gennes' Eq. VI.7 gives exact normal-mode decomposition. We should validate our simulator by checking that the Rouse mode autocorrelation functions decay exponentially with the correct τ_p ~ (N/p)² scaling.
  - **This is a stronger validation than R_g or bond-length histograms** because it tests the *dynamics* (time-dependent behavior) rather than just the *static* equilibrium structure.
  - **For the GNN:** checking whether the GNN-predicted rollout trajectories preserve the correct Rouse mode spectrum would be a *much more informative* evaluation metric than just R_g or MSD — it directly tests whether the model learned the correct relaxation dynamics across all length scales.

  > **Published reference confirming this approach:** Kremer & Grest, *J. Chem. Phys.* 92, 5057 (1990) — the original KG paper validates their model by exactly this Rouse-mode analysis. Likhtman & McLeish, *Macromolecules* 35, 6332 (2002) — extended Rouse-mode analysis to entangled chains.

### Concept 14: Friction Coefficient and Hydrodynamic Interactions (Zimm Model)
**De Gennes, §VI.1.2 and §VI.2 (pp. 167–185):**

The bead friction coefficient μ⁻¹ = 6πη₀a_H (Eq. VI.9), where η₀ is solvent viscosity and a_H is hydrodynamic radius.

**The Rouse model's weakness:** it ignores *hydrodynamic interactions* (HI) — when bead n moves, it creates a flow field that drags other beads along. Including HI (the Kirkwood/Zimm approach) changes the dynamics fundamentally:

> **Rouse (no HI):** D ~ T/(Nζ), τ_R ~ N², R ~ N^{1/2}
>
> **Zimm (with HI):** D ~ T/(η₀R), τ_Z ~ η₀R³/T ~ N^{3ν}, where ν ≈ 0.588

In 3D good solvent: τ_Z ~ N^{1.76} (vs. τ_R ~ N²)

**Status: 🟡 PARTIALLY ADOPTED — key design decision**
- **Where:** Plan §2: HOOMD-blue's `Brownian` integrator implements *Rouse-like* dynamics (no HI). The plan correctly identifies this as the "overdamped Langevin" limit.
- **What de Gennes tells us:** Our simulation is in the **Rouse regime** (no HI). This is valid for:
  - Polymer melts (where HI is screened by surrounding chains)
  - Implicit-solvent CG simulations (standard practice)
  
  But it is *not* valid for dilute polymer solutions in explicit solvent, where Zimm dynamics applies.
  
- **Recommendation:** The report should explicitly state: "Our overdamped Brownian dynamics corresponds to the Rouse model (freely draining limit), which neglects hydrodynamic interactions. This is appropriate for our coarse-grained, implicit-solvent simulation and corresponds to melt-like dynamics. The Zimm regime (non-draining limit), relevant for dilute solutions in explicit solvent, is not within our scope."

### Concept 15: Monomer MSD Subdiffusion
From Rouse theory (de Gennes §VI.1, and modern references):

> **g₁(t) = ⟨(r_n(t) − r_n(0))²⟩:**
> - t ≪ τ_R: g₁ ~ t^{1/2} (**subdiffusion** — chain connectivity constrains monomer motion)
> - t ≫ τ_R: g₁ ~ t¹ (normal diffusion — chain moves as a whole)
>
> **g₃(t) = ⟨(R_cm(t) − R_cm(0))²⟩:**
> - All t: g₃ ~ t¹ (center-of-mass always diffuses normally)
> - D_cm = k_BT / (Nγ) (diffusion coefficient scales as 1/N)

**Status: 🟡 PARTIALLY ADOPTED — critical gap for validation**
- **Where adopted:** Plan §12 lists "MSD" as a metric. [numpy_simulator.py](file:///Users/k.siddharthareddy/Documents/BASHI+OK/PINN/src/simulator/numpy_simulator.py#L150-L195) doesn't explicitly compute MSD time series yet.
- **🔴 Critical gap:** The plan mentions "MSD" but doesn't specify *what kind* of MSD. There are **three distinct MSD functions** that provide different information:
  1. **g₁(t)**: Single monomer MSD — shows subdiffusion at short times
  2. **g₂(t)**: Monomer MSD relative to center of mass — isolates internal modes
  3. **g₃(t)**: Center-of-mass MSD — tests overall diffusion
  
  The t^{1/2} subdiffusion in g₁ is a *hallmark signature of Rouse dynamics* and is one of the most stringent tests of whether the simulator is producing correct dynamics. The GNN rollout should also reproduce this subdiffusive behavior — if it doesn't, the model is not learning the correct physics even if one-step errors look small.

- **Recommendation:** Implement all three g functions as validation metrics. Add g₁(t) scaling check to the simulator validation (Month 3) and to the GNN rollout evaluation (Month 8). This is a much stronger test than just "does R_g look reasonable."

### Concept 16: Dynamic Scaling — Universality of Dynamics
**De Gennes, §VI.2 (pp. 173–185):** Just as static properties show universal scaling (R ~ N^ν), dynamic properties also show universality:

> **τ ~ R^z** where z is the dynamic exponent
>
> - Rouse: z = 2 + 1/ν (since τ_R ~ N² ~ R^{2/ν})
> - Zimm: z = 3 (since τ_Z ~ R³)

**Status: ⬜ IRRELEVANT for current scope**
- **Why:** We're in the Rouse regime and don't vary the dynamic model. The dynamic exponent z is relevant for comparing *different* dynamics models, which we don't do.

### Concept 17: Internal Friction (Cerf Friction)
**De Gennes, §VI.4 (pp. 198–204):** Beyond solvent friction, polymer chains can have *internal* friction due to conformational barriers. De Gennes shows this is irrelevant for long chains in solution (N → ∞) but can matter for short chains or chains in a solid matrix.

**Status: ⬜ IRRELEVANT**
- **Why:** We use a Kremer-Grest model with no internal friction barriers. The beads have only solvent friction (γ parameter). For our chain lengths (N=30–200), de Gennes' own analysis says internal friction is negligible.

---

## Chapter VII: Many-Chain Systems — Respiration Modes (pp. 205–218)

### Concept 18: Cooperative Diffusion in Semi-Dilute Solutions
**De Gennes, §VII.1:** In semi-dilute solutions, concentration fluctuations relax via cooperative diffusion D_coop ~ k_BT / (6πη₀ξ), where ξ is the correlation length (blob size).

**Status: ⬜ IRRELEVANT** — Single-chain simulation, no concentration fluctuations.

---

## Chapter VIII: Entanglement Effects / Reptation (pp. 219–240)

### Concept 19: Reptation — The Tube Model
**De Gennes, §VIII.2 (pp. 223–234):** For entangled polymers (long chains in a melt), de Gennes proposed the **reptation** model: a chain moves by sliding along its own contour within a "tube" formed by surrounding chains.

> **Key predictions:**
> - τ_reptation ~ N³ (vs. τ_R ~ N² for unentangled chains)
> - D_rep ~ N⁻² (vs. D_Rouse ~ N⁻¹)
> - MSD shows additional subdiffusive regimes: g₁ ~ t^{1/4} (t < τ_e), then t^{1/2} (τ_e < t < τ_R), then t^{1/2} again (τ_R < t < τ_rep), then t¹

**Status: ⬜ IRRELEVANT for current scope**
- **Why:** Reptation requires entanglements, which require long chains (N > N_e ≈ 50–85 for KG model) in a *multi-chain melt*. Our single-chain simulation cannot entangle. Even our N=200 chain (plan §4) is single-chain, so reptation does not apply.
- **If scope ever expanded:** For multi-chain systems above the entanglement length, reptation would fundamentally change the dynamics. A GNN that could learn both Rouse (unentangled) and reptation (entangled) dynamics from the same architecture would be a major result.

---

## Chapter IX: Self-Consistent Fields and Random Phase Approximation (pp. 245–264)

### Concept 20: Self-Consistent Field Theory
**De Gennes, §IX.2:** SCF methods compute chain conformations in external potentials self-consistently. The chain's density profile both *creates* and *responds to* the potential field.

**Status: ⬜ IRRELEVANT**
- **Why:** SCF is a theoretical/computational method for equilibrium structures of many-chain systems (polymer brushes, block copolymer self-assembly). Not applicable to our single-chain dynamics project.

---

## Chapter X: Polymer Statistics and Critical Phenomena (pp. 265–289)

### Concept 21: The n→0 Mapping (Polymer-Magnet Correspondence)
**De Gennes, §X.2:** De Gennes' most famous theoretical contribution: the statistics of a self-avoiding walk are *mathematically equivalent* to the n-component magnetic spin model in the limit n → 0. This maps the Flory exponent ν to the critical exponent of the magnetic correlation length.

**Status: ⬜ IRRELEVANT**
- **Why:** This is deep theoretical physics that explains *why* the scaling laws work, but doesn't add any practical equations or predictions to our simulation or GNN. It belongs in the "Further Reading" section of a report, not in the codebase.

---

## Chapter XI: Renormalization Group Ideas (pp. 290–end)

### Concept 22: Renormalization Group and the Fixed Point
**De Gennes, §XI.1:** The RG provides a systematic way to compute ν and other exponents beyond Flory's mean-field estimate. It also explains *why* universality holds — the RG fixed point is independent of microscopic details.

**Status: ⬜ IRRELEVANT** — Same as above. Theoretical foundation, not practical for our project.

---

## Cross-Cutting Concepts Not Tied to a Single Chapter

### Concept 23: The End-to-End Distance vs. Radius of Gyration Relationship
**De Gennes uses both R_F (end-to-end, Flory radius) and R_g throughout.** For a Gaussian chain: R_g² = R_ee²/6. For a self-avoiding chain, the ratio is approximately the same but with weak N-dependent corrections.

**Status: ✅ ADOPTED**
- **Where:** Plan §5 stores both `end_to_end_vector` and R_g. [numpy_simulator.py](file:///Users/k.siddharthareddy/Documents/BASHI+OK/PINN/src/simulator/numpy_simulator.py#L170-L177) computes both. Plan §12 lists both as metrics.
- **Comparison:** Rubinstein & Colby (Ch. 2) gives a more detailed treatment of the R_g / R_ee ratio and its N-dependence.

### Concept 24: Temperature-Dependent Chain Behavior
**De Gennes, throughout:** Temperature appears in three ways:
1. **As k_BT in the thermal noise** (drives Brownian motion)
2. **As the excluded volume parameter v(T)** (determines solvent quality)
3. **Near Θ (theta temperature):** v(T) → 0, chain becomes ideal

**Status: 🟡 PARTIALLY ADOPTED**
- **Where:** k_BT = 1.0 in reduced units (plan §4). Temperature is an OOD axis (plan §9, Month 9).
- **Gap:** The plan varies temperature as an OOD test but doesn't connect it to solvent quality changes. With WCA (always repulsive), changing T changes the *noise magnitude* but doesn't change the excluded-volume strength qualitatively. A truly temperature-dependent excluded-volume crossover (good → theta → poor solvent) would require a full Lennard-Jones potential (attractive + repulsive), not just WCA.
- **Recommendation:** Acknowledge in the report that our T-variation is a "kinetic temperature" variation (changing noise), not a thermodynamic solvent-quality variation. The physics is still correct but the interpretation is different from what a polymer physicist might expect.

---

## Summary: What We Should Pick Up from De Gennes

### Already Well-Covered (no action needed)
| Concept | Where in Plan/Code |
|---------|-------------------|
| Universal scaling R ~ N^ν | §1.4, §13.1, Month 10 |
| Flory exponent ν ≈ 0.588 | §2 Month 2, §12, §13.1 |
| Bead-spring chain model | Entire codebase |
| Brownian/overdamped dynamics | §2, §3, integrators.py |
| Excluded volume (WCA) | §4, forces.py |
| R_g and R_ee as observables | §5, §12, numpy_simulator.py |

### Should Be Added or Deepened
| Concept | Priority | Effort | Impact |
|---------|----------|--------|--------|
| **Rouse mode analysis for simulator validation** | 🔴 HIGH | Medium | Much stronger dynamics validation than R_g alone |
| **Rouse mode spectrum check for GNN rollout** | 🔴 HIGH | Medium | Tests whether GNN learns correct multi-scale relaxation |
| **MSD subdiffusion (g₁ ~ t^{1/2}) validation** | 🔴 HIGH | Low | Standard polymer dynamics benchmark, easy to implement |
| **Three distinct MSD functions (g₁, g₂, g₃)** | 🟡 MEDIUM | Low | More informative than a single "MSD" number |
| **Effective exponent ν_eff for finite-N chains** | 🟡 MEDIUM | Low | Prevents misinterpreting scaling study results |
| **Explicit Rouse/Zimm regime acknowledgment in report** | 🟡 MEDIUM | Zero (writing) | Strengthens physics credibility of report |
| **WCA = good-solvent-only acknowledgment** | 🟡 MEDIUM | Zero (writing) | Honest scoping in limitations section |

### Deliberately Excluded (with reasons)
| Concept | Why Irrelevant |
|---------|---------------|
| Melt screening (Ch. II) | Single-chain simulation |
| Semi-dilute blob model (Ch. III) | Single dilute chain |
| Phase separation / theta solvent (Ch. IV) | WCA is always good-solvent |
| Gelation / percolation (Ch. V) | Linear chain, no cross-links |
| Hydrodynamic interactions / Zimm (Ch. VI.2) | Implicit solvent, Rouse regime by design |
| Reptation / tube model (Ch. VIII) | Single chain, no entanglements |
| SCF theory (Ch. IX) | Equilibrium theory for many-chain systems |
| Polymer-magnet mapping / RG (Ch. X–XI) | Deep theory, no practical equations needed |

---

## Comparison with Other Key Works

| Work | What it adds beyond de Gennes | Relevance to us |
|------|------------------------------|-----------------|
| **Doi & Edwards (1986)** *Theory of Polymer Dynamics* | Rigorous derivation of Rouse/Zimm/reptation equations; detailed MSD predictions | ⭐⭐⭐ Our primary dynamics reference |
| **Rubinstein & Colby (2003)** *Polymer Physics* | Pedagogical, modern treatment; clear finite-chain-size discussions | ⭐⭐⭐ Already in plan §1.4 |
| **Kremer & Grest (1990)** *J. Chem. Phys.* | Original KG bead-spring model paper; validates via Rouse modes | ⭐⭐⭐⭐ Our model's origin |
| **Sanchez-Gonzalez et al. (2020)** GNS paper | Noise injection for rollout stability | ⭐⭐⭐ Already in plan §10, model #2 |
| **Satorras et al. (2021)** EGNN paper | E(n)-equivariant GNN architecture | ⭐⭐⭐ Already in plan §10, model #6 |
| **Sharma & Fink (2026)** Dynami-CAL GraphNet | Momentum-conserving GNN via edge-local frames | ⭐⭐⭐ Already in plan §11, model #5 |
