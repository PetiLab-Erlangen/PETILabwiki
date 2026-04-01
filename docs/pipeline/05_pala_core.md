# 5. PALA Core Processing

## Purpose

This page describes the computational heart of the pipeline, implemented mainly in:

- `3_compute_fast_slow_tracks/run_PALA_single_general.m`
- `3_compute_fast_slow_tracks/get_pala_params_csv.m`
- `3_compute_fast_slow_tracks/pala_toolbox/PALA_functions/SVDfilter.m`
- `3_compute_fast_slow_tracks/pala_toolbox/ULM_toolbox/ULM_localization2D.m`
- `3_compute_fast_slow_tracks/pala_toolbox/ULM_toolbox/ULM_tracking2D.m`
- `3_compute_fast_slow_tracks/pala_toolbox/ULM_toolbox/ULM_Track2MatOut.m`

The PALA core transforms block-wise ultrasound data into:

- localized microbubble positions
- tracked trajectories
- structural maps
- directional maps
- velocity maps

---

## High-level sequence inside `run_PALA_single_general`

For each dataset and mode, the function performs the following sequence:

1. read DICOM metadata
2. compute frame rate and spatial scale `lambda`
3. list all `block_*.mat` files
4. load the correct parameter row from the CSV
5. build the `ULM` parameter struct
6. loop over blocks
7. optionally apply TGC
8. apply SVD filtering
9. optionally apply Butterworth temporal filtering
10. localize microbubble candidates
11. link detections into tracks
12. aggregate tracks across blocks
13. convert tracks into output matrices
14. save `.mat` outputs and rendered figures

---

## Spatial and temporal calibration

### Frame rate

Frame rate is extracted from the DICOM metadata. This gives the temporal scale:

\[
\Delta t = \frac{1}{f_{\text{frame}}}
\]

Without this value, any velocity estimate would remain in arbitrary frame units.

### Wavelength-like spatial scale

The code estimates:

\[
\lambda = \frac{c}{f}
\]

with \(c \approx 1540 \, \text{m/s}\) and \(f\) obtained from the transducer metadata. If not available, the fallback used in the code is:

\[
\lambda = \frac{1540}{2500}
\]

in the unit convention of the current implementation.

This `lambda` is later used to convert track-based velocity outputs into physical units.

---

## The ULM parameter structure

The `ULM` struct built in `run_PALA_single_general.m` contains the processing assumptions. Important fields include:

- `numberOfParticles`
- `size`
- `scale`
- `res`
- `SVD_cutoff`
- `max_linking_distance`
- `min_length`
- `fwhm`
- `max_gap_closing`
- `interp_factor`
- `TGCMax`, `TGCMin`
- `LocMethod`
- `ButterCuttofFreq`
- `lambda`

This struct is passed into the downstream localization and tracking functions.

---

## Step 1. Optional TGC

TGC compensates for depth-dependent attenuation. Echoes from deeper tissue tend to be weaker, so a depth-dependent gain can help equalize detectability.

Conceptually, if \(I(z)\) decays with depth, TGC applies a gain \(g(z)\) such that:

\[
I_{\text{TGC}}(z) = g(z)\, I(z)
\]

This can reduce depth bias but can also amplify noise. That is why TGC remains optional and configurable by CSV.

---

## Step 2. SVD filtering

This is one of the most important steps in the whole pipeline.

Let the spatiotemporal data matrix be reshaped into \(X\). Singular value decomposition writes:

\[
X = U \Sigma V^{T}
\]

where:

- \(U\): spatial singular vectors
- \(V\): temporal singular vectors
- \(\Sigma\): singular values ranked by energy

### Physical intuition

Tissue clutter is usually:

- high energy
- spatially coherent
- slowly varying over time

Microbubbles are usually:

- sparse
- less coherent
- more transient

Therefore, tissue often dominates the first singular modes. By suppressing a chosen range of singular components, the pipeline attenuates clutter while preserving flow-related signal.

In practice, the filtered signal is conceptually:

\[
X_{\text{filt}} = \sum_{k \in \mathcal{K}} \sigma_k\, u_k v_k^{T}
\]

where \( \mathcal{K} \) is the retained set after the SVD cutoff rule.

Bad SVD settings can cause:

- under-filtering: clutter remains
- over-filtering: true microbubble signal is removed

---

## Step 3. Optional Butterworth temporal filtering

If enabled, the code designs a second-order bandpass Butterworth filter:

```matlab
[but_b, but_a] = butter(2, ULM.ButterCuttofFreq / (framerate/2), 'bandpass');
```

Applied along time, it removes:

- very slow drift
- temporal high-frequency noise

The idea is to preserve temporal dynamics consistent with flow while rejecting signals outside the chosen frequency band.

In frequency-domain language, if \(H(\omega)\) is the filter response and \(X(\omega)\) the temporal spectrum, the filtered signal becomes:

\[
Y(\omega) = H(\omega)\, X(\omega)
\]

This can be useful, but if the cutoff is too aggressive it may suppress genuine slow dynamics.

---

## Step 4. Localization

Localization is performed on the magnitude image:

```matlab
MatTracking = ULM_localization2D(abs(IQ_filt), ULM);
```

### Why localization is super-resolution

A single microbubble is smaller than the effective ultrasound point spread function (PSF), so it does not appear as one perfect pixel. Instead, it appears blurred. ULM gains resolution by estimating the center of this blurred pattern more precisely than the pixel spacing.

If the observed image is modeled as:

\[
I(x,z) = \left[\delta(x-x_0,z-z_0) * \text{PSF}(x,z)\right] + \eta(x,z)
\]

then localization aims to estimate \( (x_0,z_0) \) from the blurred response.

That is the core physical reason why ULM can beat the diffraction-limited appearance of conventional ultrasound.

---

## Step 5. Tracking

After localization, the detections are linked across frames using motion constraints. The tracking stage reconstructs the trajectory of each bubble candidate.

A track is a sequence of positions:

\[
\mathbf{p}_1, \mathbf{p}_2, \ldots, \mathbf{p}_N
\]

with

\[
\mathbf{p}_i = (z_i, x_i)
\]

From this, the path length and direction can be estimated.

Tracking is governed by parameters such as:

- maximum linking distance
- minimum allowed track length
- maximum gap closing

These are crucial because dense detections can lead to ambiguous associations.

---

## Step 6. Reconstruction into MatOut maps

After tracks are assembled, the code computes:

- `MatOut`
- `MatOut_zdir`
- `MatOut_vel`

using `ULM_Track2MatOut`.

### `MatOut`

This is the structural accumulation map. It reflects where tracked signal passed and is the basis for vessel-density-like visualization.

### `MatOut_zdir`

This encodes directional information, especially the sign or orientation of axial flow contribution.

### `MatOut_vel`

This is computed first in the internal scale of the track representation and then converted in your code with:

```matlab
MatOut_vel = MatOut_vel * ULM.lambda;
```

This is important: the final velocity map is not left in arbitrary tracking units.

---

## Velocity interpretation

If a track moves by spatial increment \( \Delta s \) over \( \Delta t \), then:

\[
v = \frac{\Delta s}{\Delta t}
\]

In your implementation, the super-resolved track coordinates and frame rate define the internal displacement/time relation, and multiplication by `lambda` performs the physical scaling expected by the rest of the pipeline.

This is why the map later becomes interpretable as **mm/s-like velocity output** instead of just “pixels per frame”.

---

## Saved outputs

For each branch and dataset, the function saves representative outputs such as:

- `<filename>example_tracks.mat`
- `<filename>example_Alltracks.mat`
- `<filename>example_matouts.mat`

These contain tracks, MatOut maps, `llx`, `llz`, `ULM`, and `lambda`.

This is extremely useful because the metrics stage can reuse these products without rerunning the expensive reconstruction.

---

## Scientific meaning of this stage

This stage is where the ultrasound cine becomes a **vascular representation**. Before this point, the data are mainly image intensities over time. After this point, the data are:

- localized trajectories
- spatial density maps
- direction maps
- velocity maps

That is the critical transition from raw imaging to quantitative microvascular inference.

---

## Summary

The PALA core is the most physics-heavy stage of the pipeline. It combines:

- DICOM-derived calibration
- optional depth compensation
- low-rank clutter rejection
- optional temporal filtering
- sub-resolution localization
- trajectory reconstruction
- map formation in physical space

Everything downstream—ROI metrics, glomeruli analysis, correlations, and the dashboard—depends on the quality of this stage.
