# 8. Metrics and Equations

## Purpose

This stage transforms the reconstructed ULM outputs — localization maps, velocity fields, and tracked trajectories — into quantitative vascular biomarkers. Each metric is anchored to a published physical or physiological definition; this page documents both the implementation and its scientific basis, so the pipeline can be reported in full in a methods section.

Main implementation files:

- `4_compute_ROIs_and_metrics/run_kidney_metrics_batch.m`
- `4_compute_ROIs_and_metrics/helpers/run_kidney_metrics_single_dataset.m`
- `4_compute_ROIs_and_metrics/helpers/ulm_metrics.m`
- `4_compute_ROIs_and_metrics/helpers/compute_ulm_metrics.m`
- `4_compute_ROIs_and_metrics/helpers/tracks_to_grid.m`
- `4_compute_ROIs_and_metrics/helpers/tortuosity_over_tracks.m`
- `4_compute_ROIs_and_metrics/helpers/compute_tortuosity_VTI.m`
- `4_compute_ROIs_and_metrics/helpers/voxel_tortuosity_map.m`
- `4_compute_ROIs_and_metrics/helpers/compute_dispersity_distance_metric_roi.m`

The stage produces three output granularities:

| Scale | File pattern | Content |
|-------|-------------|---------|
| ROI | `*_kidney_metrics_summary.csv` | One row per ROI per dataset |
| Voxel | `*_kidney_metrics_RAWVALUES.csv` | Per-pixel values for Python aggregation |
| Track | `*_kidney_tortuosity_tracks.csv` | Per-track tortuosity values before aggregation |

---

## Physical context: what ULM outputs represent

Before describing each metric, it is important to recall what the upstream pipeline produces. After SVD clutter filtering, sub-resolution localization, and multi-frame tracking, three maps are available over the super-resolved grid:

- **`MatOut(x,z)`**: the spatial accumulation map. Each pixel holds the count of microbubble localizations that fell within that pixel across the entire acquisition. Formally:

$$
D(x,z) = \sum_{t=1}^{T} \sum_{k} \mathbf{1}\bigl[(x_k^t, z_k^t) \in \text{pixel}(x,z)\bigr]
$$

where the inner sum runs over all localizations $k$ detected at frame $t$. This is the ULM density map as defined in Hingot et al. (2019) and reviewed in Christensen-Jeffries et al. (2020).

- **`MatOut_vel(x,z)`**: the mean microbubble speed at each pixel, in mm/s after `lambda` scaling. Computed from inter-frame displacements of tracked bubbles passing through that voxel.

- **Tracks**: sequences of super-resolved positions $\{\mathbf{p}_i\} = \{(z_i, x_i)\}$ linked across frames. These carry both geometric (tortuosity, dispersity) and kinematic (speed) information.

```
  Raw IQ frames
       |
   SVD filter  ────────► clutter removed
       |
  Localization ────────► sub-px bubble positions (x,z,t)
       |
   Tracking    ────────► trajectories {p1,...,pN}
       |
  Track2MatOut ────────► D(x,z)   MatOut_vel(x,z)
       |
  Metric stage ────────► density · velocity · tortuosity · dispersity · PAF · Phi
```

---

## 8.1 ROI Area

For every ROI mask containing $N_\text{pix}$ pixels with physical spacings $\Delta x$ and $\Delta z$ (mm):

$$
A_\text{ROI} = N_\text{pix} \cdot \Delta x \cdot \Delta z \qquad [\text{mm}^2]
$$

This area normalizes density and glomerular density to physically meaningful units independent of image resolution or zoom level.

---

## 8.2 Microbubble Density and Microvascular Blood Volume (MVBV)

### Physical definition

The density map $D(x,z)$ is the ULM analog of microvascular blood volume (MBV) in conventional contrast-enhanced ultrasound (CEUS). In CEUS, MBV is proportional to the plateau signal intensity $A$ of the wash-in curve (Wei et al. 1998). In ULM, the same information is encoded spatially: pixels that are repeatedly traversed by microbubbles accumulate higher counts, directly reflecting the time-integrated presence of flowing contrast.

$$
\bar{D}_\text{ROI} = \frac{1}{N_\text{pix}} \sum_{(x,z)\,\in\,\text{ROI}} D(x,z) \qquad [\text{counts/pixel}]
$$

Normalized to physical area:

$$
\rho = \frac{\sum_{(x,z)\,\in\,\text{ROI}} D(x,z)}{A_\text{ROI}} \qquad [\text{counts/mm}^2]
$$

### Microvascular Blood Volume (MVBV)

MVBV is the fraction of the ROI area that was visited by at least one microbubble:

$$
\text{MVBV}_\text{vox} = \#\{(x,z) \in \text{ROI} : D(x,z) > 0\}
$$

$$
\text{MVBV}_\text{area} = \text{MVBV}_\text{vox} \cdot \Delta x \cdot \Delta z \qquad [\text{mm}^2]
$$

This is the standard ULM density output as defined in Hingot et al. (2019) and the PALA benchmark (Heiles et al. 2022).

### Important note

$\rho$ is not a histological vessel count. It is a ULM-derived density surrogate proportional to the time-integrated microbubble transit signal per unit area. Because acquisition duration, bubble concentration, and frame rate all affect the absolute count, $\rho$ should be used for within-study relative comparisons unless acquisition parameters are standardized.

**References:** Hingot et al. *Sci Rep* 2019 [6]; Christensen-Jeffries et al. *Ultrasound Med Biol* 2020 [7]; Heiles et al. *Nat Biomed Eng* 2022 [5]; Wei et al. *Circulation* 1998 [8].

---

## 8.3 Velocity

### Physical definition

Microbubble speed is estimated from inter-frame displacements along each track. For a bubble at positions $(z_i, x_i)$ and $(z_{i+1}, x_{i+1})$ separated by time interval $\Delta t = 1/f_\text{frame}$:

$$
\|\mathbf{v}_i\| = \frac{\sqrt{(\Delta z_i)^2 + (\Delta x_i)^2}}{\Delta t} = \frac{\|\mathbf{p}_{i+1} - \mathbf{p}_i\|}{\Delta t}
$$

In code this is `hypot(vz, vx)` where `vz` and `vx` are the pre-computed velocity components scaled by `lambda`. This is the standard ULM velocimetry formula used in Errico et al. (2015) and Christensen-Jeffries et al. (2015).

The ROI-level mean speed is:

$$
\bar{v}_\text{ROI} = \frac{1}{N} \sum_{i=1}^{N} \|\mathbf{v}_i\| \qquad [\text{mm/s}]
$$

where the sum runs over all valid velocity samples (positive voxels) within the ROI mask applied to `MatOut_vel`.

### Physical interpretation

In the microcirculation, red blood cell velocity in capillaries ranges from ~0.2 mm/s to ~2 mm/s, and in larger cortical vessels up to ~10 mm/s. ULM velocity estimates reflect bubble advection by the local flow field, so $\bar{v}$ is a physiologically grounded flow-dynamics surrogate — not a direct Doppler measurement, but capturing the same hemodynamic information with spatial resolution inaccessible to conventional Doppler.

**References:** Errico et al. *Nature* 2015 [3]; Christensen-Jeffries et al. *IEEE TMI* 2015 [4].

---

## 8.4 Perfused Area Fraction (PAF)

### Physical definition

PAF is defined as the fraction of ROI pixels visited by at least one microbubble:

$$
\text{PAF} = \frac{\#\{(x,z) \in \text{ROI} : D(x,z) > 0\}}{N_\text{pix}}
$$

This corresponds to the binary perfused-area fraction reported in Lowerison et al. (2025) and Dencks & Schmitz (2023). The threshold of one count (any microbubble passage) is the minimal positive threshold: a pixel is considered perfused if at least one bubble traversed it during the acquisition.

In the conventional CEUS literature (Rubin et al. 1994), PAF corresponds to the "Fractional Moving Blood Volume" — the fraction of pixels showing Doppler-positive signal.

### Implementation note

An earlier version of the code used the median density as a threshold, producing a "supra-median density fraction" rather than true PAF. This was corrected:

```matlab
% TRUE PAF: fraction of ROI voxels with at least 1 localization
S.roi(r).paf = mean(counts > 0);
% Original supra-median metric preserved under a distinct name
S.roi(r).high_density_fraction = mean(counts >= max(1, round(prctile(counts, 50))));
```

By construction, `high_density_fraction <= PAF` always holds, because requiring $D \geq \text{median}$ is strictly more demanding than requiring $D > 0$.

### Biological interpretation

In healthy renal cortex, PAF reflects how densely the glomerular and peri-tubular capillary network is populated with flowing bubbles. A reduced PAF may indicate microvascular rarefaction, reduced perfusion, or incomplete bubble delivery — as seen in renal impairment or fibrosis.

**References:** Lowerison et al. *eLife* 2025 [10]; Dencks & Schmitz *Z Med Phys* 2023 [11]; Rubin et al. *Radiology* 1994 [13].

---

## 8.5 Perfusion Proxy (Phi — density × velocity)

### Physical rationale

In CEUS, volumetric blood flow $Q$ through a region is related to the wash-in curve parameters by (Wei et al. 1998):

$$
Q \propto A \cdot \beta
$$

where $A$ is plateau signal intensity (proportional to microvascular blood volume) and $\beta$ is the wash-in rate constant (proportional to mean microbubble velocity). In ULM, $D(x,z)$ approximates $A$ and $v(x,z)$ approximates the velocity field, yielding a local perfusion proxy:

$$
\Phi(x,z) = D(x,z) \cdot v(x,z)
$$

Two ROI-level estimators are implemented:

**Map-based (preferred):**

$$
\Phi_\text{map} = \frac{1}{N_\text{pix}} \sum_{(x,z)\in\text{ROI}} D(x,z) \cdot v(x,z)
$$

This computes the voxelwise product before averaging, preserving spatial co-variation between density and velocity.

**Track-based:**

$$
\Phi_\text{tracks} = \bar{D} \cdot \bar{v}
$$

This is only equivalent to $\Phi_\text{map}$ if density and velocity are spatially uncorrelated — i.e., $\mathbb{E}[DV] = \mathbb{E}[D]\cdot\mathbb{E}[V]$. In renal cortex this assumption is unlikely (denser arcuate regions also carry faster flow), so $\Phi_\text{map}$ is preferred.

$\Phi$ has arbitrary units and should be reported as a relative perfusion index, not an absolute flow rate.

**References:** Wei et al. *Circulation* 1998 [8].

---

## 8.6 Tortuosity — Distance Index (DI)

### Physical definition

The Distance Index (DI) is the classical arc-to-chord tortuosity ratio introduced by Bullitt et al. (2003). For a track with $N$ points $\mathbf{p}_1, \ldots, \mathbf{p}_N$:

$$
L_\text{arc} = \sum_{i=1}^{N-1} \|\mathbf{p}_{i+1} - \mathbf{p}_i\| = \sum_{i=1}^{N-1} \sqrt{(\Delta z_i)^2 + (\Delta x_i)^2}
$$

$$
L_\text{chord} = \|\mathbf{p}_N - \mathbf{p}_1\| = \sqrt{(z_N - z_1)^2 + (x_N - x_1)^2}
$$

$$
\text{DI} = \frac{L_\text{arc}}{L_\text{chord}}
$$

DI = 1 for a perfectly straight path; DI > 1 for any curved trajectory. Tracks with $L_\text{chord} = 0$ (closed loops or stationary detections) are excluded. The ROI-level index is the mean over all qualifying tracks entering the ROI.

**Validation status: CORRECT** — exact match to Bullitt et al. (2003).

### Physical interpretation

DI captures path-level geometric complexity. In renal microvasculature, elevated DI may reflect glomerular capillary loops, pathological kinking, or microangiopathy. A DI of 1.05 is mildly tortuous; values above ~1.2 indicate clinically significant winding.

**References:** Bullitt et al. *IEEE Trans Med Imaging* 2003 [1].

---

## 8.7 Tortuosity — Angular Change (deg/mm, SOAM-adapted)

### Physical definition

This metric quantifies how rapidly the flow direction changes per unit path length — a measure of local curvature integrated over the vessel. It is an adaptation of the Sum of Angles Measure (SOAM) from Bullitt et al. (2003).

For consecutive segment vectors with directions $\theta_i = \text{atan2}(\Delta z_i, \Delta x_i)$, the signed angular change between segments $i$ and $i+1$ is wrapped to $[-\pi, \pi]$:

$$
\delta\theta_i = \bigl[(\theta_{i+1} - \theta_i) + \pi \;\mathrm{mod}\; 2\pi\bigr] - \pi
$$

The total accumulated turning normalized by path length:

$$
\tau_\text{deg/mm} = \frac{180}{\pi} \cdot \frac{\displaystyle\sum_{i} |\delta\theta_i|}{L_\text{arc}}
$$

### Difference from canonical SOAM

Canonical SOAM uses the absolute tangent angle of each point relative to a fixed axis. This implementation uses direction changes between consecutive segment vectors — appropriate for ULM discrete tracks (one localization per frame), where spline-based tangent estimation would introduce interpolation artifacts. For smooth microvascular paths, both formulations converge.

In publications: label as **"angular-change tortuosity (deg/mm)"**, not SOAM.

**Validation status: DEVIATION (minor)** — physically correct and appropriate for discrete ULM tracks.

**References:** Bullitt et al. *IEEE Trans Med Imaging* 2003 [1].

---

## 8.8 Tortuosity — VTI (Vessel Tortuosity Index, ULM-adapted)

### Published formula (Khansari et al. 2017)

The Vessel Tortuosity Index combines arc length, angular variability, inflection structure, and local arc-to-chord ratios:

$$
\text{VTI} = 0.1 \times \frac{L_\text{arc} \times \sigma_\theta \times N_\text{crit} \times \overline{DM}}{L_\text{chord}}
$$

| Symbol | Definition |
|--------|-----------|
| $L_\text{arc}$ | total arc length |
| $L_\text{chord}$ | endpoint chord length |
| $\sigma_\theta$ | SD of tangent angles along the vessel |
| $N_\text{crit}$ | number of critical points (direction-reversal events) |
| $\overline{DM}$ | mean arc-to-chord ratio between consecutive inflection points |
| $0.1$ | empirical scaling constant (values fall in perceptible range) |

### Implementation — step by step

**Step 1** — segment directions:

$$
\theta_i = \text{atan2}(\Delta z_i,\, \Delta x_i), \quad i = 1, \ldots, N-1
$$

**Step 2** — angular SD ($\sigma_\theta$):

$$
\sigma_\theta = \text{std}(\theta_1, \ldots, \theta_{N-1})
$$

Uses segment-vector directions (see Section 8.8.1 for justification).

**Step 3** — discrete curvature:

$$
\kappa_i = \frac{\Delta\theta_i}{\bar{s}_i}, \quad \Delta\theta_i = \theta_{i+1} - \theta_i, \quad \bar{s}_i = \tfrac{1}{2}(s_i + s_{i+1})
$$

**Step 4** — inflection points (for $\overline{DM}$):

Inflection points are zero-crossings of $\Delta\theta$ (sign changes of curvature):

$$
\text{inflection at } i \iff \Delta\theta_i \cdot \Delta\theta_{i+1} < 0
$$

Between consecutive inflection points $a$ and $b$:

$$
DM_{ab} = \frac{\displaystyle\sum_{i=a}^{b-1} s_i}{\|\mathbf{p}_b - \mathbf{p}_a\|}
$$

$\overline{DM}$ is the mean over all such segments. If fewer than two inflection points exist, the fallback is the global DI (whole-track arc-to-chord ratio), consistent with Khansari's original fallback.

**Step 5** — critical points ($N_\text{crit}$):

Direction-reversal events, detected as sign changes of $\Delta\theta$:

$$
N_\text{crit}^\text{raw} = \#\{i : \Delta\theta_i \cdot \Delta\theta_{i+1} < 0\}
$$

Floor applied to prevent degenerate collapse for monotonically curved tracks:

$$
N_\text{crit} = \max(N_\text{crit}^\text{raw},\; 1)
$$

Without this floor, VTI = 0 for any monotonically curving track (e.g., a quarter-circle arc) regardless of how curved it is — a physically incorrect result.

**Step 6** — composite index:

$$
\text{VTI} = 0.1 \times \frac{L_\text{arc} \times \sigma_\theta \times N_\text{crit} \times \overline{DM}}{L_\text{chord}}
$$

### 8.8.1 Justification for segment-based $\sigma_\theta$

Khansari's original work used retinal angiography, where vessels are imaged as dense continuous pixel chains and spline-based point-level tangent estimation is appropriate. ULM tracks are fundamentally discrete: one localization per frame (~20–500 ms intervals), with localization uncertainty dependent on SNR. Under these conditions:

1. Fitting a spline to estimate point-level tangents would introduce interpolation artifacts not present in the original data.
2. Segment-vector directions are the highest-resolution angular information naturally available from tracking.
3. For smooth microvascular paths (radius of curvature much larger than inter-localization spacing), segment-vector $\sigma_\theta$ converges to the continuous tangent-angle $\sigma_\theta$ as sampling density increases.

This adaptation should be documented in publications as **"VTI (ULM-adapted, segment-vector $\sigma_\theta$)"** with explicit reference to the discrete track structure.

### Bugs corrected vs. original implementation

Three bugs were identified and fixed:

| Bug | Original | Fixed |
|-----|---------|-------|
| Missing 0.1 scaling | `VTI = (L_arc x sigma x N x DM) / L_chord` | `VTI = 0.1 x (...)` |
| Critical point detection | Sign changes of `diff(curvature)` (2nd derivative peaks) | Sign changes of `diff(theta)` (1st derivative reversals) |
| No floor on $N_\text{crit}$ | Absent — VTI = 0 for monotone arcs | `N_crit = max(N_crit, 1)` |

**Validation status: DIVERGES from Khansari (2017)** in $\sigma_\theta$ definition (segment vs. point tangents) — justified by ULM data structure. Label as **VTI-adapted** in publications.

**References:** Khansari et al. *Biomed Opt Express* 2017 [2]; Bullitt et al. *IEEE Trans Med Imaging* 2003 [1].

---

## 8.9 Voxel-wise Tortuosity Map

Rather than one tortuosity value per track, the pipeline also produces a spatial tortuosity map $T(x,z)$ by aggregating track-level DI values over local voxel neighborhoods:

$$
T(x,z) = \frac{1}{K(x,z)} \sum_{k \in \mathcal{K}(x,z)} \text{DI}_k
$$

where $\mathcal{K}(x,z)$ is the set of tracks passing through the neighborhood of pixel $(x,z)$.

This map reveals spatial heterogeneity in vessel geometry that ROI-level averages would obscure — for example, the tortuous glomerular capillary loops in the cortex produce locally elevated $T$ values invisible in a whole-cortex mean.

Voxels with $K(x,z) < K_\text{min}$ are masked as NaN to avoid unstable local estimates.

---

## 8.10 Dispersity (Circular Statistics)

### Physical motivation

Microvascular flow is not isotropic: cortical vessels tend to align radially (toward the medulla), while medullary vasa recta descend almost vertically. Dispersity quantifies how disordered or aligned the local flow directions are — a property invisible to speed or tortuosity metrics.

### Mathematical formulation

For tracks with orientations $\theta_k$, the axial double-angle trick maps antipodally symmetric directions to unique angles (a vessel going right and one going left have the same orientation):

$$
\phi_k = 2\theta_k \pmod{2\pi}
$$

The length-weighted circular mean vector:

$$
\bar{C} = \frac{\sum_k w_k \cos\phi_k}{\sum_k w_k}, \qquad
\bar{S} = \frac{\sum_k w_k \sin\phi_k}{\sum_k w_k}
$$

where $w_k = L_\text{arc}^{(k)}$ weights longer tracks more. The mean resultant length:

$$
R = \sqrt{\bar{C}^2 + \bar{S}^2} \in [0,\,1]
$$

Dispersity:

$$
\text{Dispersity} = 1 - R
$$

- $R \to 1$ (Dispersity $\to 0$): all tracks aligned — coherent flow.
- $R \to 0$ (Dispersity $\to 1$): isotropic orientations — fully disordered flow.

The circular mean orientation:

$$
\mu_\text{ROI} = \frac{1}{2}\,\text{atan2}(\bar{S},\,\bar{C})
$$

### Spatial dispersity (deg/mm)

$$
\tau_\text{dispersity} = \frac{\overline{|\theta_k - \mu_\text{ROI}|}}{r_\text{RMS}} \qquad [\text{deg/mm}]
$$

This gives a rate of directional disorder per unit spatial extent, useful for comparing ROIs of different physical sizes.

### Comparison to Denis et al. (2023)

Denis et al. define dispersity as the fraction of track segments not within ±20° of the dominant direction — a threshold-based approach. The $1-R$ formulation here is the rigorous circular-statistics analog (Mardia & Jupp 2000): threshold-free, more sensitive to graded directional disorder, and more robust when no single dominant direction exists (as in glomerular capillary tufts).

**Validation status: CORRECT** — textbook circular statistics, more rigorous than Denis et al. (2023).

**References:** Denis et al. *eBioMedicine* 2023 [9]; Mardia & Jupp *Directional Statistics* 2000 [12].

---

## 8.11 Track Count and Support Metrics

Several metrics depend on how many tracks enter a ROI or neighborhood. These counts directly determine statistical stability:

- Tortuosity estimates from fewer than 5 tracks are unreliable.
- Dispersity from fewer than 10 tracks may show spurious alignment.
- Voxel-level maps should be masked where track count is below $K_\text{min}$.

Track count per ROI is always exported and should be reported alongside metrics in any publication to allow readers to assess reliability.

---

## 8.12 Glomeruli Metrics

The ROI summary table includes glomerular quantities from the glomeruli detection stage (Section 9):

- `glomeruli_nb`: number of detected glomeruli in the ROI
- `glomeruli_density_glom_per_cm2`: count normalized to ROI area
- `glomeruli_area_cm2`: total area occupied by detected glomerular structures

Glomerular density has an established histological reference range (~5–10 glom/mm² in cortex by histology), providing an independent validation anchor for the pipeline.

---

## 8.13 Summary: Metric–Physics–Reference Table

| Metric | Physical meaning | Formula | Validation | Key reference |
|--------|----------------|---------|------------|---------------|
| Density $\rho$ | Time-integrated bubble transit per area | $\sum D / A_\text{ROI}$ | CORRECT | Hingot 2019 [6] |
| MVBV | Perfused area in mm² | $\#\{D>0\} \cdot \Delta x \Delta z$ | CORRECT | Hingot 2019 [6] |
| Speed $\bar{v}$ | Mean microbubble advection speed | $\overline{\|\mathbf{v}\|}$ | CORRECT | Errico 2015 [3] |
| PAF | Fraction of ROI pixels with ≥1 bubble | $\#\{D>0\}/N_\text{pix}$ | CORRECT (after fix) | Lowerison 2025 [10] |
| $\Phi_\text{map}$ | Relative perfusion index | $\overline{D \cdot v}$ | DEVIATION (minor) | Wei 1998 [8] |
| DI | Arc-to-chord tortuosity | $L_\text{arc}/L_\text{chord}$ | CORRECT | Bullitt 2003 [1] |
| deg/mm | Angular-change tortuosity | $\sum|\delta\theta|/L_\text{arc}$ | DEVIATION (minor) | Bullitt 2003 [1] |
| VTI | Composite tortuosity index | $0.1 \times L_\text{arc}\,\sigma_\theta\,N_\text{crit}\,\overline{DM}/L_\text{chord}$ | DIVERGES (adapted) | Khansari 2017 [2] |
| Dispersity | Directional disorder $1-R$ | Axial circular resultant length | CORRECT | Mardia & Jupp 2000 [12] |

---

## 8.14 Multi-scale output design

The pipeline preserves information at three nested scales:

```
  Voxel scale  ─── D(x,z), v(x,z), T(x,z)  ─── spatial heterogeneity, maps
       |
  Track scale  ─── DI_k, VTI_k, theta_k     ─── individual trajectory geometry
       |
  ROI scale    ─── mean, std per ROI         ─── biomarker summary for statistics
```

This design is important because:

1. Voxel maps allow visualization of regional heterogeneity within an ROI.
2. Track-level values allow distribution analysis (histogram of DI) rather than just means.
3. ROI summaries support cohort-level statistical testing and correlation with clinical variables.

Discarding intermediate scales would lose diagnostically relevant information — for example, the variance of DI within a ROI may be as informative as its mean for detecting patchy disease.

---

## Scientific meaning of the metrics stage

This stage is where the pipeline stops being an image-reconstruction tool and becomes a **biomarker extraction framework**. Each metric captures a distinct and physically interpretable dimension of microvascular biology:

- **Density** → how much vascular territory is populated with flowing bubbles
- **MVBV / PAF** → what fraction of the ROI is perfused at all
- **Velocity** → how fast contrast moves through those vessels
- **DI / deg/mm / VTI** → how geometrically complex the vessel paths are
- **Dispersity** → how organized or disordered the flow directions are
- **Glomeruli density** → how many discrete filtration units are detectable

Together, these metrics describe the kidney as a multidimensional vascular phenotype — enabling differentiation between compartments (cortex vs. medulla), conditions (healthy vs. disease), and time points (longitudinal follow-up).

---

## References

1. Bullitt E, Gerig G, Pizer SM, Lin W, Aylward SR. Measuring tortuosity of the intracerebral vasculature from MRA images. *IEEE Trans Med Imaging.* 2003;22(9):1163–71. DOI: 10.1109/TMI.2003.816964
2. Khansari MM, O'Neill W, Lim J, Shahidi M. Method for quantitative assessment of retinal vessel tortuosity in optical coherence tomography angiography applied to sickle cell retinopathy. *Biomed Opt Express.* 2017;8(8):3796–3806. DOI: 10.1364/BOE.8.003796
3. Errico C, Pierre J, Pezet S, et al. Ultrafast ultrasound localization microscopy for deep super-resolution vascular imaging. *Nature.* 2015;527:499–502. DOI: 10.1038/nature16066
4. Christensen-Jeffries K, Browning RJ, Tang M-X, Dunsby C, Eckersley RJ. In vivo acoustic super-resolution and super-resolved velocity mapping using microbubbles. *IEEE Trans Med Imaging.* 2015;34(2):433–440. DOI: 10.1109/TMI.2014.2359650
5. Heiles B, Chavignon A, Hingot V, et al. Performance benchmarking of microbubble-localization algorithms for ultrasound localization microscopy. *Nat Biomed Eng.* 2022;6(5):605–616. DOI: 10.1038/s41551-021-00824-8
6. Hingot V, Errico C, Heiles B, et al. Microvascular flow dictates the compromise between spatial resolution and acquisition time in ULM. *Sci Rep.* 2019;9:2456. DOI: 10.1038/s41598-018-38349-x
7. Christensen-Jeffries K, Couture O, Dayton PA, et al. Super-resolution Ultrasound Imaging. *Ultrasound Med Biol.* 2020;46(4):865–891. DOI: 10.1016/j.ultrasmedbio.2019.11.013
8. Wei K, Jayaweera AR, Firoozan S, et al. Quantification of myocardial blood flow with ultrasound-induced destruction of microbubbles. *Circulation.* 1998;97(5):473–483. DOI: 10.1161/01.CIR.97.5.473
9. Denis L, Bodard S, Hingot V, et al. Sensing ultrasound localization microscopy for the visualization of glomeruli in living rats and humans. *eBioMedicine.* 2023;91:104578. DOI: 10.1016/j.ebiom.2023.104578
10. Lowerison MR, Sekaran NVC, Dong Z, et al. Longitudinal awake imaging of mouse deep brain microvasculature with super-resolution ULM. *eLife.* 2025;14:e95168. DOI: 10.7554/eLife.95168
11. Dencks S, Schmitz G. Ultrasound localization microscopy. *Z Med Phys.* 2023;33(3):292–308. DOI: 10.1016/j.zemedi.2023.02.004
12. Mardia KV, Jupp PE. *Directional Statistics.* Chichester: Wiley; 2000. DOI: 10.1002/9780470316979
13. Rubin JM, Bude RO, Carson PL, et al. Power Doppler US: a potentially useful alternative to mean frequency-based color Doppler US. *Radiology.* 1994;190(3):853–856. DOI: 10.1148/radiology.190.3.8115639
