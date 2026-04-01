# 12. Dashboard

## Purpose

The Streamlit dashboard provides an interactive front end to the pipeline outputs. It is implemented in:

- `5_visualization/python_dashboard/app.py`
- launched from `5_visualization/run_python_dashboard.m`

This stage does not reconstruct ULM itself. Instead, it makes the results explorable.

---

## What the dashboard reads

The app is designed to read:

- MATLAB-generated figures
- Python combined tables from `python_analysis_outputs`
- merged ULM + external variable tables
- plotter functions from `ulm_thesis.plotters`

It supports both a flat output folder and a `tables/` subfolder layout.

---

## Main dashboard capabilities

- participant selection
- interactive FAST/SLOW comparison
- browsing of cohort figures and montages
- loading of combined cohort CSVs
- interactive correlation exploration
- participant-level boxplots and summaries

This is particularly valuable when the study has multiple participants and many metrics.

---

## Why it matters scientifically

A static PDF can show only a few selected views. The dashboard makes it possible to:

- inspect an outlier participant
- switch between ROIs
- compare FAST and SLOW instantly
- explore correlations interactively

This makes the pipeline far more practical for iterative scientific interpretation.

---

## Launch logic

The MATLAB launcher writes a BAT file that starts Streamlit with the correct arguments:

```text
streamlit run app.py -- --baseDir <baseDir> --analysisDir <analysisDir>
```

The dashboard then opens in the browser, typically at:

```text
http://localhost:8501
```

---

## Summary

The dashboard is the final interpretation layer of the refactor. It turns the pipeline from a batch-processing system into an exploratory analysis environment.
