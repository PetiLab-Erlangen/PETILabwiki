# 9. Glomeruli Analysis

## Purpose

This stage extends the standard ROI workflow by adding a more localized renal microvascular analysis centered on glomeruli-like structures or glomeruli-related overlays.

Relevant files include:

- `4_compute_ROIs_and_metrics/helpers/compute_glomeruli_in_rois.m`
- `4_compute_ROIs_and_metrics/helpers/compute_glomeruli_overlay_data_single_dataset.m`
- `4_compute_ROIs_and_metrics/helpers/build_glomeruli_overlay_data_batch.m`
- `5_visualization/helpers/make_ulm_cohort_glomeruli_montage_from_overlay.m`

---

## Why glomeruli matter

The kidney is not only a network of generic vessels. It contains highly specialized microvascular units, and glomeruli are among the most biologically important. A pipeline that can connect ULM-derived signal to glomeruli-related structure has greater translational relevance.

This is one reason your refactor goes beyond generic super-resolution reconstruction.

---

## How glomeruli outputs enter the pipeline

The metrics stage writes summary columns such as:

- `glomeruli_nb`
- `glomeruli_density_glom_per_cm2`
- `glomeruli_area_cm2`

This means glomeruli-related outputs are treated as first-class quantitative variables, not only as decorative overlays.

---

## Conceptual metric definitions

If \(N_{\text{glom}}\) is the number of detected glomeruli-like objects inside an ROI of area \(A\), then the area-normalized density is:

\[
\rho_{\text{glom}} = \frac{N_{\text{glom}}}{A}
\]

In the exported summary table, the normalized unit is reported per cm² for interpretability:

\[
\rho_{\text{glom}}^{(\text{cm}^{-2})} = \frac{N_{\text{glom}}}{A_{\text{cm}^2}}
\]

This provides a localized structural measure that is distinct from general track density.

---

## Overlay workflow

The overlay utilities exist so that glomeruli-related detections can be visualized together with the underlying ULM map and the anatomical ROI. This is extremely useful for:

- quality control
- spotting implausible detections
- preparing poster or paper figures
- comparing FAST and SLOW behavior visually

The batch exporter produces a file such as:

```text
glomeruli_overlay_batch_summary.csv
```

which can then be consumed by cohort-visualization functions.

---

## Biological interpretation

A glomeruli-related count or density is not identical to histology, but it can provide a structured imaging-derived surrogate of how many localized microvascular objects are present in the selected compartment.

This is particularly interesting in the cortex, where glomerular microvascular features are most relevant.

---

## Why this stage is valuable

From a documentation standpoint, this page signals that your refactor is not only about generic PALA processing. It explicitly targets **renal microvascular phenotyping**.

That elevates the pipeline from:

> “track bubbles and make maps”

to:

> “extract biologically meaningful renal microvascular biomarkers”

---

## Summary

The glomeruli stage adds localized structural analysis on top of the general ULM metrics. It enriches the renal interpretation of the pipeline and provides outputs that can be correlated with other imaging and clinical variables.
