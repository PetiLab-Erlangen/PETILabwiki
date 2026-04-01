# 13. Outputs and Folder Structure

## Purpose

This page summarizes what the pipeline writes to disk and why the folder structure matters.

The refactor is intentionally organized so that expensive stages can be reused without rerunning everything from zero.

---

## Top-level study layout

A study root contains many participants:

```text
<baseDir>/
├── ULM_NTX_01/
├── ULM_NTX_02/
├── ULM_NTX_03/
└── ...
```

---

## Typical participant layout after processing

```text
ULM_NTX_01/
├── blocks/
│   ├── block_001.mat
│   ├── block_002.mat
│   └── ...
├── result/
│   └── Motion_corrected_data.mat
├── ULM/
│   ├── ROIs/
│   └── fast/
│       ├── metrics/
│       ├── plots/
│       └── ...
│   └── slow/
│       ├── metrics/
│       ├── plots/
│       └── ...
├── python_analysis_outputs/
│   ├── summary_all.csv
│   ├── raw_all.csv
│   ├── tracks_all.csv
│   └── ...
```

---

## Important output families

### Blocks
Used as the canonical input for the ULM core.

### MatOut files
Used to avoid rerunning localization and tracking when only metrics or figures need to be regenerated.

### Metrics CSVs
Used by both MATLAB figures and Python post-analysis.

### Plots and panels
Used for QC, presentations, and publications.

### Combined Python tables
Used for cohort summaries, correlations, and dashboard interaction.

---

## Why this structure is good

The structure supports a modular workflow:

- rerun only metrics if the tracks already exist
- rerun only Python if the CSVs already exist
- rerun only the dashboard if the combined tables already exist

This saves a large amount of time during iterative development and figure polishing.

---

## Summary

The output structure is not just an organizational choice. It is part of the computational design of the refactor, enabling reuse, debugging, and cohort-scale reproducibility.
