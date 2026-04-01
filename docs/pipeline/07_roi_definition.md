# 7. ROI Definition

## Purpose

This stage defines the anatomical compartments used for all regional biomarker extraction. In your refactor, the main kidney ROIs are:

- cortex
- medulla
- whole kidney

The ROI information is stored under:

```text
ULM/ROIs/
```

and is reused by both FAST and SLOW metric computation.

---

## Where ROIs appear in the code

Relevant logic appears during both block building and metric computation:

- ROI polygon selection during block generation
- ROI loading inside `run_kidney_metrics_single_dataset.m`
- ROI overlay figures generated during the metrics stage

This design is important because the same anatomical ROIs are used consistently across multiple outputs.

---

## Why ROIs are essential

Without ROIs, the pipeline could only produce whole-image summaries. That would be much less meaningful biologically because the kidney is spatially heterogeneous.

Different renal compartments have different vascular architecture and physiological roles. Therefore, a metric such as density or tortuosity should not only be asked globally, but regionally:

\[
\text{metric}_{\text{cortex}} \neq \text{metric}_{\text{medulla}} \neq \text{metric}_{\text{whole kidney}}
\]

In other words, ROIs convert a generic image-processing workflow into a renal biomarker workflow.

---

## ROI masks in the metric stage

If the image grid is defined over \((x,z)\), then each ROI is represented by a binary mask:

\[
M_r(x,z) =
\begin{cases}
1, & (x,z) \in \text{ROI}_r \\
0, & \text{otherwise}
\end{cases}
\]

This mask is used to extract values from maps such as:

- `MatOut`
- `MatOut_vel`
- voxel tortuosity maps
- glomeruli overlays

For any map \(Q(x,z)\), the ROI-specific values are:

\[
Q_r = \{ Q(x,z) \;|\; M_r(x,z)=1 \}
\]

and summary metrics are then computed from that set.

---

## ROI reuse across FAST and SLOW

A very good feature of your refactor is that the anatomical ROIs are shared across FAST and SLOW processing. That means differences observed between FAST and SLOW are more likely to reflect reconstruction dynamics rather than inconsistent anatomy.

This improves the validity of comparisons such as:

- cortex_FAST vs cortex_SLOW
- medulla_FAST vs medulla_SLOW

---

## Visualization role

The metrics stage produces ROI overlay images on top of perfusion-like or structural backgrounds. These figures have two purposes:

1. quality control of the anatomical segmentation
2. visual communication for reports and publications

Because of this, ROI definition is not only a numerical step. It is also a **validation step**.

---

## Biological interpretation

### Cortex
The cortex is expected to contain a rich microvascular network. Metrics derived here may be especially sensitive to changes in superficial perfusion and glomerulus-associated regions.

### Medulla
The medulla has different vascular organization and may show distinct density, directionality, and flow patterns.

### Whole kidney
The whole-kidney ROI provides an integrated view and is often helpful for cohort-level summaries.

---

## Summary

The ROI stage is where the pipeline becomes anatomically grounded. It ensures that later density, velocity, tortuosity, dispersity, and glomeruli metrics are not just image-wide numbers, but **region-specific biomarkers** tied to real kidney compartments.
