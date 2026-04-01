# 1. Dataset and Inputs

## Purpose

This stage defines **what a valid dataset looks like** for the pipeline and how participant folders are discovered. The refactor is built around a directory structure in which a single study root contains many datasets, each one processed in the same way.

The batch driver responsible for scanning datasets is:

```matlab
run_all_participants_2D_ULM.m
```

---

## Expected dataset naming

The code searches for directories matching:

```regex
^ULM_[A-Za-z]+_\d+$
```

Examples:

- `ULM_NTX_01`
- `ULM_NTX_10`
- `ULM_GRAN_02`

This convention is important because it lets the orchestrator:

1. detect valid participant folders automatically
2. sort them numerically
3. derive the study code used for parameter selection

If no folder matches this pattern, the pipeline stops with an error.

---

## What lives inside a dataset folder

A participant folder evolves as the pipeline runs. Early in the workflow it mainly contains raw input data; later it accumulates blocks, ULM outputs, metrics, figures, and analysis tables.

A typical processed dataset will contain or generate folders such as:

```text
ULM_NTX_01/
├── blocks/
├── result/
├── ULM/
│   ├── ROIs/
│   ├── fast/
│   │   ├── metrics/
│   │   └── plots/
│   └── slow/
│       ├── metrics/
│       └── plots/
```

---

## Raw imaging input

The current refactor is designed around ultrasound cine input read from DICOM. The DICOM contains not only the image frames, but also metadata used later for quantitative conversion, especially:

- frame timing
- cine rate or frame rate
- transducer frequency
- ultrasound region information

Those metadata matter because ULM is fundamentally a **spatiotemporal estimation problem**. A displacement in pixels per frame only becomes a physical velocity after we know the spatial and temporal sampling:

\[
v = \frac{\Delta s}{\Delta t}
\]

where \( \Delta s \) depends on spatial calibration and \( \Delta t \) depends on frame rate.

---

## DICOM-derived physical quantities

Inside `run_PALA_single_general.m`, the code extracts:

- `framerate`
- `lambda`

The wavelength-like spatial scale is inferred as:

\[
\lambda = \frac{c}{f}
\]

with

- \(c \approx 1540 \, \text{m/s}\) in soft tissue
- \(f\) the transducer center frequency

In the code, if frequency is unavailable, a fallback is used:

\[
\lambda = \frac{1540}{2500}
\]

in the units expected by the pipeline implementation.

This matters because your track reconstruction and velocity maps are later converted into physical units using `lambda`.

---

## Block input format

The PALA processing stage does not read the raw DICOM directly frame by frame during reconstruction. Instead, the earlier block-building stage converts the cine into a set of `.mat` block files containing a 3D variable:

```matlab
IQ
```

with dimensions approximately:

\[
[n_z, \; n_x, \; n_t]
\]

where:

- \(n_z\) = depth samples
- \(n_x\) = lateral samples
- \(n_t\) = frames in the block

This is the true numerical input consumed by the ULM core.

---

## Study code and mode awareness

A dataset name such as `ULM_NTX_01` carries study identity (`NTX`) that is reused downstream to select the correct numerical parameters from the CSV.

This is architecturally important because the pipeline is not a single fixed reconstruction: it is a **parameterized reconstruction framework**.

---

## Why input structure matters

A quantitative ULM pipeline is only as reliable as the consistency of its inputs. If participant folders are not standardized, the batch driver cannot guarantee:

- reproducible ordering
- correct study-code assignment
- consistent storage of outputs
- safe reuse of metrics and figures

This dataset stage therefore acts as the foundation for cohort-scale processing.

---

## Inputs and outputs

### Inputs
- study root selected by the user
- raw participant folders
- DICOM cine and metadata

### Outputs
- ordered dataset list
- validated folder naming
- study code and participant identity ready for downstream stages

---

## Summary

This stage defines the pipeline’s contract with the data. It answers:

- **what counts as one participant**
- **where the raw data live**
- **how the study is identified**
- **which physical metadata are available for conversion into real units**

Without this layer, later metrics such as velocity in mm/s or density in counts/mm² would not be trustworthy.
