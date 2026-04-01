# Code Map

## Orchestrator
- `run_all_participants_2D_ULM.m`

---

## Configuration
- `0_config/config_ULM_paths_general.m`
- `0_config/Standard_ULM_Parameters.CSV`

---

## Block building
- `2_build_blocks/build_blocks_without_mc_ui.m`
- `0_config/src/blocks/build_blocks_with_srus_mc.m`
- helpers: ROI selection, dual-panel detection

---

## Motion correction
- `0_config/src/srus/run_motion_correction_srus.m`
- outputs: `result/Motion_corrected_data.mat`

---

## PALA core
- `3_compute_fast_slow_tracks/run_PALA_single_general.m`
- `get_pala_params_csv.m`
- `SVDfilter.m`
- `ULM_localization2D.m`
- `ULM_tracking2D.m`
- `ULM_Track2MatOut.m`

---

## FAST / SLOW
- mode-specific params via CSV
- outputs in `ULM/fast/` and `ULM/slow/`

---

## ROI + Metrics
- `4_compute_ROIs_and_metrics/run_kidney_metrics_batch.m`
- helpers:
  - `tracks_to_grid.m`
  - `tortuosity_over_tracks.m`
  - `voxel_tortuosity_map.m`
  - `compute_dispersity_distance_metric_roi.m`
  - `compute_glomeruli_in_rois.m`

---

## Visualization
- `5_visualization/helpers/*`
- panels, montages, overlays

---

## Python analysis
- `5_visualization/python_analysis/run_post_metrics.py`
- `ulm_thesis/{io,report,plotters}.py`

---

## Dashboard
- `5_visualization/python_dashboard/app.py`

---

## Outputs
- Blocks, MatOut, metrics CSVs, plots
- `python_analysis_outputs/` (combined tables)

This map links every documentation page to concrete implementation files, enabling direct traceability from concept to code.
