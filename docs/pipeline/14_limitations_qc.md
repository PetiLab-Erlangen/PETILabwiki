# 14. Limitations and QC

## Purpose

This page summarizes the main limitations of the current refactor and the most important quality-control checks that should accompany interpretation.

---

## 1. 2D limitation

The current workflow reconstructs a 2D imaging plane, not a full 3D vascular volume. Therefore, track geometry and directionality are plane-dependent.

A vessel that appears straight in one plane may be curved in three dimensions.

---

## 2. SVD sensitivity

SVD cutoff selection is critical. If the cutoff is too low, clutter remains; if it is too high, true bubble signal is suppressed.

This directly affects:

- density
- velocity
- continuity of tracks
- interpretability of FAST and SLOW maps

---

## 3. Tracking ambiguity in dense regions

When local detections are very dense, linking errors become more likely. This may inflate or distort:

- path length
- tortuosity
- local direction maps

Track-level QC is therefore important.

---

## 4. Motion sensitivity

If motion correction is not used, residual organ or probe motion can mimic vessel structure. Even with motion correction, imperfect registration may remain.

---

## 5. ROI dependence

ROI metrics are only as anatomically meaningful as the ROI masks themselves. Misdrawn or inconsistent ROIs can bias every downstream regional summary.

---

## 6. Slow-track noise

SLOW reconstructions are often more vulnerable to residual noise and over-fragmentation, especially when the underlying data quality is weak or the filtering is suboptimal.

This is one reason side-by-side FAST/SLOW visual QC is essential.

---

## Recommended QC checks

### A. Check block previews
Confirm the selected frame range and crop are anatomically meaningful.

### B. Check structural maps
Verify that `MatOut` looks vascular and not like diffuse clutter.

### C. Check velocity maps
Look for implausible extreme values or large noisy backgrounds.

### D. Check ROI overlays
Ensure the cortex, medulla, and whole-kidney masks align with the anatomy.

### E. Check track-level tables
Look for very short, fragmented, or extreme-tortuosity tracks.

### F. Check cohort montages
Outlier participants often become obvious only when shown together.

---

## Interpretation advice

The safest way to interpret the pipeline is to combine:

1. quantitative summary tables
2. representative maps
3. track-level plausibility
4. ROI overlay QC
5. cohort context

No single metric should be interpreted in isolation.

---

## Summary

The refactor is already a strong end-to-end ULM framework, but like all advanced imaging pipelines, it requires disciplined quality control. The best interpretation comes from treating metrics, maps, and anatomy as a combined system rather than independent outputs.
