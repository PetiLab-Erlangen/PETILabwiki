# 0. Configuration and Paths

## Purpose

This stage initializes the pipeline and makes the refactor portable across computers, users, and study roots. In the current codebase, the central configuration entry point is:

```matlab
cfg = config_ULM_paths_general();
```

implemented in:

```text
REFRACTOR/0_config/config_ULM_paths_general.m
```

The job of this layer is **not** to process ultrasound data directly. Its job is to guarantee that all later stages see a coherent environment: correct code paths, a selected study root, remembered preferences, and access to parameter files.

---

## Main files in this stage

- `0_config/config_ULM_paths_general.m`
- `0_config/Standard_ULM_Parameters.CSV`
- `0_config/src/config/pipeline_config.m`
- `0_config/src/blocks/build_blocks_with_srus_mc.m`
- `0_config/src/blocks/build_blocks_without_mc.m`
- `0_config/src/blocks/build_blocks_mc_only.m`
- `0_config/src/srus/run_motion_correction_srus.m`

---

## What the configuration function actually does

### 1. Detects the code root automatically

The function derives the project root from the location of `config_ULM_paths_general.m` itself. This is important because the code is meant to be portable and should not require hard-coded paths to the refactor repository.

Conceptually:

\[
\text{projectRoot} = \text{parent}(\text{folder containing config\_ULM\_paths\_general})
\]

This makes the code robust when copied to another machine or moved inside a new directory tree.

### 2. Lets the user choose the study base folder

The code then opens a folder picker and asks the user to select the **data root** that contains all participant folders. In your workflow, that folder contains datasets with names such as:

```text
ULM_NTX_01
ULM_NTX_02
ULM_GRAN_01
```

This selected folder becomes `baseDir`.

### 3. Remembers the last selected study root

The function uses MATLAB preferences (`getpref`, `setpref`) so that the next time the pipeline is launched, the last successful study root can be offered again. This is very useful in a lab workflow where the same project root is used repeatedly.

### 4. Adds the code to the MATLAB path in a filtered way

The config function adds the refactor folders to the MATLAB path while excluding clutter such as:

- `.git`
- `__pycache__`
- virtual environments
- Python output folders
- IDE folders such as `.vscode`

This reduces path noise and avoids package warnings.

### 5. Exposes the main handles to the rest of the pipeline

The output structure `cfg` provides, at minimum:

- `cfg.baseDir`
- `cfg.projectRoot`
- code helper paths
- path to the parameter CSV

That structure is then consumed by `run_all_participants_2D_ULM.m`.

---

## Parameter file: Standard_ULM_Parameters.CSV

The parameter CSV is the bridge between the workflow logic and the numerical settings used by the PALA processing stage. It stores mode- and study-specific parameters such as:

- number of particles
- SVD cutoff
- localization method
- FWHM
- minimum track length
- linking distance
- Butterworth frequency cutoffs
- TGC settings

This means the numerical behavior of the ULM core is **data-driven** rather than hard-coded.

That is important because FAST and SLOW processing use different assumptions about the expected motion regime.

---

## Why this matters scientifically

In a ULM pipeline, the reconstruction quality depends strongly on the coherence between:

- acquisition metadata
- frame rate
- spatial calibration
- filtering parameters
- tracking constraints

This configuration stage is where the pipeline guarantees that all later modules start from the same assumptions.

In practical terms, it prevents three common problems:

1. using the wrong data root
2. mixing code from different branches or folders
3. running FAST/SLOW with inconsistent parameter sets

---

## Inputs and outputs

### Inputs
- Refactor repository on disk
- Study root selected by the user
- Parameter CSV

### Outputs
- initialized MATLAB path
- selected `baseDir`
- reusable configuration struct `cfg`

---

## Relationship to the rest of the pipeline

This stage is called immediately by the batch orchestrator:

```matlab
cfg = config_ULM_paths_general();
baseDir = cfg.baseDir;
```

Everything else depends on that `baseDir`. For this reason, configuration is the **true stage zero** of the pipeline.

---

## Practical note

The configuration layer does not compute microvascular biomarkers itself, but it controls the conditions under which all biomarkers are later computed. In that sense, it is part of the reproducibility backbone of the full workflow.
