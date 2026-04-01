# 10. Visualization Outputs

## Purpose

The visualization layer converts reconstruction and metrics into interpretable figures for quality control, cohort comparison, and publication use.

Relevant files include:

- `5_visualization/helpers/make_ulm_bmode_metric_panel_single_dataset.m`
- `5_visualization/helpers/make_ulm_wholekidney_metric_montages.m`
- `5_visualization/helpers/make_ulm_cohort_glomeruli_montage_from_overlay.m`
- `5_visualization/helpers/render_matouts_like_maincode.m`

---

## Main visualization products

### 1. Single-dataset panels
These show one participant at a time, often with B-mode underlay, ROI contours, and one or more metric maps.

### 2. Whole-kidney cohort montages
These place multiple participants into the same visual template so FAST/SLOW and participant-to-participant patterns can be compared.

### 3. Publication panels
These are meant to be cleaner, more paper-ready exports.

### 4. Glomeruli montages
These visualize glomeruli-related overlays at cohort scale.

---

## Why visualization is not optional

A quantitative pipeline still needs visual validation. Numerical outputs alone cannot fully reveal:

- motion-correction failure
- implausible vessel geometry
- ROI misalignment
- noisy slow-track reconstructions
- unphysical directional maps

Visualization is therefore part of the scientific validation chain.

---

## Types of maps shown

### Structural map (`MatOut`)
Shows where tracks accumulate and therefore where the microvascular network is reconstructed.

### Direction map (`MatOut_zdir`)
Shows the signed or oriented flow structure.

### Velocity map (`MatOut_vel`)
Shows local mean speed of tracks, scaled into the intended physical units.

### Tortuosity map
Shows local curvature complexity of the reconstructed vascular paths.

---

## Why B-mode underlays are useful

The ULM map alone shows vascular structure, but not necessarily organ anatomy. Overlaying ULM outputs on B-mode provides anatomical context:

- kidney boundary
- cortex/medulla relation
- ROI plausibility
- clinician-friendly interpretation

This is very important for translational work, especially when communicating with an audience that is less focused on code details.

---

## Summary

The visualization layer turns the pipeline into something inspectable, interpretable, and publishable. It is the bridge between raw quantitative outputs and human judgment.
