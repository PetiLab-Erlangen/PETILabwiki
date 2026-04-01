# 11. Python Post-analysis

## Purpose

After MATLAB has finished reconstruction and metric extraction, Python performs cohort-scale aggregation, cleaning, correlation analysis, and figure generation.

Main files:

- `5_visualization/run_python_post_metrics.m`
- `5_visualization/python_analysis/run_post_metrics.py`
- `5_visualization/python_analysis/ulm_thesis/io.py`
- `5_visualization/python_analysis/ulm_thesis/report.py`
- `5_visualization/python_analysis/ulm_thesis/plotters.py`

---

## Why Python is used here

MATLAB is excellent for signal processing and direct integration with the original PALA code. Python is especially useful for:

- joining many CSV files
- cleaning long-form and wide-form tables
- correlation analysis
- generating report-ready statistical figures
- building the dashboard

This produces a hybrid architecture:

- MATLAB = reconstruction engine
- Python = data science and reporting layer

---

## MATLAB-to-Python handoff

The MATLAB wrapper `run_python_post_metrics.m` calls:

```text
python_analysis/run_post_metrics.py
```

with arguments such as:

- `--baseDir`
- `--outDir`
- optional `--excel`

The default Python output folder is:

```text
<baseDir>/python_analysis_outputs
```

---

## What `run_post_metrics.py` does

The script:

1. scans the study root
2. finds participant folders
3. reads summary, raw, and track CSVs
4. concatenates them into cohort tables
5. writes combined CSVs
6. cleans extreme or invalid values
7. generates figures and tables through the report module

Typical output files include:

- `summary_all.csv`
- `raw_all.csv`
- `tracks_all.csv`
- `raw_all_clean.csv`
- figures and tables inside subfolders

---

## Participant discovery logic

The Python IO layer supports both:

- direct `ULM_*` participant folders
- alternative layouts such as `NTX*`

This is a very useful compatibility feature because it makes the downstream analysis more robust to folder-layout variation.

---

## Cleaning logic

The Python layer performs cleaning steps before final analysis. For example, very high tortuosity outliers can be clipped or filtered using quantile rules. This is important because voxel- and track-level distributions can contain unstable extreme values.

Conceptually, if \(q_{0.995}\) is the 99.5th percentile, a cleaning rule may retain:

\[
T \le q_{0.995}
\]

This does not change the raw exports, but it improves the interpretability of cohort-level plots.

---

## Correlation analysis

The Python layer is where ULM outputs are related to other variables, for example:

- CEUS parameters
- clinical measures
- external spreadsheet values

The general form is:

\[
r = \operatorname{corr}(X, Y)
\]

where \(X\) may be a ULM metric and \(Y\) an external variable such as eGFR or a CEUS-derived quantity.

This stage is therefore where the imaging pipeline becomes multimodal and hypothesis-driven.

---

## Report generation

The `ulm_thesis/report.py` module uses Matplotlib, Seaborn, and SciPy to produce paper-like figures and save both PNG and PDF versions. This is ideal for thesis, poster, and manuscript workflows.

---

## Summary

The Python post-analysis stage takes the rich but still fragmented per-dataset MATLAB outputs and turns them into a coherent cohort-level statistical product. It is the main layer for synthesis, comparison, and interpretation beyond one participant at a time.
