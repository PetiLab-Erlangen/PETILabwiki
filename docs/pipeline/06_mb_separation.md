# 6. Microbubble Separation

## Purpose

Your refactor contains an explicit `mbSeparation` configuration inside `run_PALA_single_general.m`. This stage is not yet a fully standalone page in the original scaffold, but it is conceptually important because it formalizes how different motion components can be separated before or during interpretation.

The relevant fields currently configured in the `ULM` struct include:

```matlab
ULM.mbSeparation.enable = true;
ULM.mbSeparation.dx_mm = lambda;
ULM.mbSeparation.dz_mm = lambda;
ULM.mbSeparation.dt_s = 1/frameRate;
ULM.mbSeparation.speedEdges_mm_s = [0 0.5 2 10];
ULM.mbSeparation.dirMode = 'bidirectional';
ULM.mbSeparation.overlapFrac = 0.30;
ULM.mbSeparation.taperAlpha = 0.15;
ULM.mbSeparation.minEnergyFrac = 0.01;
ULM.mbSeparation.apodizeSpaceTime = true;
ULM.mbSeparation.normalizeInput = false;
ULM.mbSeparation.returnComplex = false;
```

---

## Why this matters

In dense contrast-enhanced ultrasound, not every temporal-spatial fluctuation should be treated as a single homogeneous bubble population. Different motion components can overlap in the same imaging region, especially when:

- multiple bubbles pass simultaneously
- velocity ranges differ strongly
- directional flow is mixed
- bright and weak trajectories coexist

Separation logic helps the pipeline preserve more meaningful motion subspaces instead of collapsing everything into one representation.

---

## Physical interpretation of the parameters

### Spatial and temporal step sizes

\[
\Delta x = \text{dx\_mm}, \qquad \Delta z = \text{dz\_mm}, \qquad \Delta t = \text{dt\_s}
\]

These define the physical grid in which motion is interpreted.

### Speed edges

The configured edges:

\[
[0,\;0.5,\;2,\;10] \; \text{mm/s}
\]

define velocity intervals or candidate motion regimes. Conceptually, they partition the signal into speed bands.

### Bidirectional mode

`dirMode = 'bidirectional'` means the separation logic is aware that motion can occur in more than one dominant direction, which is realistic in vascular trees.

### Overlap fraction and tapering

These parameters regulate how sharply or smoothly neighboring motion bands interact. A hard split would create discontinuities; tapered overlap allows more stable decomposition.

---

## Relation to FAST and SLOW

This microbubble-separation concept is related to, but not identical with, the global FAST/SLOW branch split. FAST/SLOW is the top-level reconstruction branch. `mbSeparation` is a lower-level motion decomposition configuration that can support or refine how signal components are handled internally.

So the conceptual hierarchy is:

1. global branch: FAST or SLOW
2. internal motion separation rules within that branch

---

## Why this is useful for your wiki

Even if the current implementation is still evolving, documenting `mbSeparation` is valuable because it shows that the refactor is not only a direct port of a standard PALA workflow. It is a framework that is being prepared for more explicit motion-subpopulation analysis.

That makes the pipeline more future-proof for:

- capillary vs non-capillary decomposition
- directional flow sub-analysis
- denser bubble regimes
- advanced temporal masking

---

## Summary

This stage formalizes how the pipeline can distinguish different motion populations in physical units. In a renal ULM context, that is important because real microvascular flow is heterogeneous, and not all trajectories should be interpreted as coming from the same dynamic compartment.
