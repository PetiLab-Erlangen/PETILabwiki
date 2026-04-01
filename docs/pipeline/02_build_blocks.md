# 2. Block Building

## Purpose

The block-building stage converts raw cine data into manageable 3D chunks that the ULM core can process efficiently. In the refactor, the main interactive no-motion-correction entry point is:

```matlab
build_blocks_without_mc_ui.m
```

There is also an SRUS-based branch for motion-corrected workflows.

---

## Main files in this stage

- `2_build_blocks/build_blocks_without_mc_ui.m`
- `2_build_blocks/helpers/detect_dual_panel.m`
- `2_build_blocks/helpers/get_contrast_gray_block.m`
- `2_build_blocks/helpers/pick_frame_range_ui.m`
- `2_build_blocks/helpers/pick_roi_interactive.m`
- `2_build_blocks/helpers/pick_roi_on_best_bmode.m`
- `2_build_blocks/helpers/save_roi_preview_images.m`
- `0_config/src/blocks/build_blocks_with_srus_mc.m`

---

## Why block building exists

A full cine sequence may contain many hundreds or thousands of frames. Running localization and tracking over the entire sequence in one shot is often memory-heavy and less robust for batch processing. The block strategy solves this by partitioning time into smaller windows.

If the cine contains \(N\) frames and the chosen block size is \(B\), the number of buffers is approximately:

\[
N_{\text{buffers}} = \left\lfloor \frac{N}{B} \right\rfloor
\]

Each buffer becomes one file:

```text
block_001.mat
block_002.mat
...
```

This provides:

- bounded memory usage
- easier debugging
- stage-wise restart capability
- explicit temporal chunking for PALA

---

## What happens in the no-MC UI workflow

### 1. Frame-range selection

The user chooses the start and end frame using an interactive cine viewer. The code then adjusts the final range so that it is compatible with the block size. This prevents incomplete trailing blocks.

### 2. Best B-mode frame selection

Within the chosen time range, the code identifies a representative bright B-mode frame. This frame is used for ROI drawing because it gives the user an anatomically meaningful view of the kidney.

### 3. ROI drawing

A polygon ROI is drawn interactively. That ROI is used to crop or mask the later block data so the pipeline focuses on the relevant anatomical region.

### 4. Block export

The cine is converted to grayscale or IQ-compatible block data and written to `.mat` files. Metadata sidecar files are also stored, for example:

- `frame_range.mat`
- `roi_mask.mat`

---

## Dual-panel handling

Your helper `detect_dual_panel.m` exists because some acquisitions may include a dual-panel display, for example B-mode plus contrast information side-by-side. The block builder must identify this correctly so the correct image region is used.

This matters because accidental use of the wrong panel would contaminate all downstream calculations.

---

## Output data model

Each block contains an `IQ` array with shape:

\[
IQ \in \mathbb{R}^{n_z \times n_x \times n_t}
\]

or, depending on the acquisition, complex-valued or real-valued data after conversion. The PALA stage later interprets the third axis as time.

The spatial axes later used for reconstruction are represented with vectors such as:

- `llx`
- `llz`

These define the physical coordinate system in which maps and metrics are visualized.

---

## Why ROI selection already appears here

Even though ROI-level metrics are computed later, the block-building stage already introduces region focus because:

- it reduces irrelevant background
- it stabilizes later localization
- it reduces computational burden
- it preserves a consistent anatomical frame for later ROI reuse

In other words, this is the first stage where the workflow becomes kidney-specific rather than generic ultrasound processing.

---

## Scientific interpretation

Block building does not compute perfusion or density directly, but it strongly influences both. A poor choice of frame range or crop can bias:

- apparent vessel density
- track continuity
- map completeness
- ROI coverage

Therefore, block quality is part of the quantitative chain, not just a file-format convenience.

---

## Inputs and outputs

### Inputs
- raw cine / DICOM frames
- selected frame interval
- interactively drawn ROI

### Outputs
- `blocks/block_###.mat`
- `frame_range.mat`
- `roi_mask.mat`
- ROI preview images

---

## Summary

This stage transforms the raw acquisition into the canonical numerical input of the pipeline. It decides **what temporal segment is analyzed**, **which anatomy is included**, and **how the cine is chunked** for the ULM core.
