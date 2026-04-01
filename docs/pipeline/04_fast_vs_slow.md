# 4. FAST and SLOW Tracking

## Purpose

A distinctive feature of your refactor is that the vascular reconstruction is not treated as a single homogeneous process. Instead, the pipeline runs two hemodynamic branches:

- **FAST**
- **SLOW**

These are stored separately under:

```text
ULM/fast/
ULM/slow/
```

and are run through the same structural stages with different parameter settings.

---

## Why split into FAST and SLOW?

The central biological idea is that the kidney contains vascular compartments with different flow regimes. If all tracks are pooled together, subtle differences between high-velocity and low-velocity components can be blurred.

The FAST/SLOW split gives the pipeline the ability to preserve hemodynamic heterogeneity.

In general terms:

- **FAST** tends to emphasize higher-velocity, more rapidly moving contrast trajectories
- **SLOW** tends to preserve slower microcirculatory trajectories that may represent finer vessel beds or capillary-like flow

---

## Where the split is implemented

The batch driver launches PALA in mode-specific branches:

- `run_PALA_single_general(..., mode="fast", ...)`
- `run_PALA_single_general(..., mode="slow", ...)`

The actual mode-dependent behavior comes from:

```matlab
P = get_pala_params_csv(study, mode, csvPath, SizeOfBloc(3));
```

So the split is controlled through the parameter CSV rather than by hard-coding two completely separate pipelines.

---

## What differs between FAST and SLOW

The modes can differ in parameters such as:

- `numberOfParticles`
- `SVD_cutoff`
- `max_linking_distance`
- `min_length`
- `fwhm`
- `max_gap_closing`
- Butterworth settings
- optional TGC behavior

This means FAST and SLOW are not only labels applied after the fact; they are **two differently tuned reconstructions**.

---

## Physical interpretation

Suppose a track has displacement \( \Delta s \) over time interval \( \Delta t \). Its mean speed is:

\[
v = \frac{\Delta s}{\Delta t}
\]

If the processing constraints are tuned for larger expected displacements, the algorithm becomes more permissive toward faster motion. If tuned for smaller displacements, it becomes more sensitive to slower motion but may reject long jumps.

That is why separate parameter sets are scientifically meaningful.

---

## Why this matters for kidney imaging

In renal microvascular imaging, not all vessels behave the same way. Large or less resistive pathways can produce different apparent dynamics than very fine, slow microvascular compartments.

By reconstructing FAST and SLOW separately, the pipeline can later compare:

- density_FAST vs density_SLOW
- velocity_FAST vs velocity_SLOW
- tortuosity_FAST vs tortuosity_SLOW
- cortex_FAST vs cortex_SLOW
- medulla_FAST vs medulla_SLOW

This makes the analysis much more informative than a single pooled map.

---

## Folder logic

Each mode gets its own output directory, typically containing:

- example track files
- MatOut maps
- ROI metrics
- plots
- summary CSVs

This mode isolation is also useful for debugging. If one branch looks biologically implausible, it can be inspected without affecting the other.

---

## Important caution

FAST and SLOW should not automatically be interpreted as two perfectly separated biological vessel classes. They are **algorithmic flow regimes** shaped by acquisition physics, frame rate, filtering, and tracking constraints.

So the proper interpretation is:

> FAST and SLOW are complementary reconstructions that highlight different dynamic subspaces of the same vascular acquisition.

---

## Inputs and outputs

### Inputs
- same participant blocks
- same dataset identity
- mode-specific parameter rows from the CSV

### Outputs
- `ULM/fast/*`
- `ULM/slow/*`

---

## Summary

This stage is one of the strengths of your refactor. It turns a single ULM reconstruction into a **dual-regime hemodynamic analysis**, allowing microvascular heterogeneity to be studied explicitly instead of being averaged away.
