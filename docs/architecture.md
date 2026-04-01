# Architecture

## Design

A **modular, staged architecture** orchestrated by:

```matlab
run_all_participants_2D_ULM.m
```

### Principles
- **Modularity**: each stage is restartable
- **Reproducibility**: standardized folders & CSV parameters
- **Hybrid compute**: MATLAB (physics) + Python (statistics)
- **Multi-scale outputs**: voxel → ROI → cohort

---

## Modules

1. Configuration (`config_ULM_paths_general.m`)
2. Dataset discovery (`ULM_*` folders)
3. Block building (`blocks/block_*.mat`)
4. Motion correction (optional, SRUS)
5. PALA core:
   - SVD filtering
   - optional Butterworth
   - localization
   - tracking
   - MatOut maps
6. FAST/SLOW branches (CSV-driven parameters)
7. ROI mapping (cortex/medulla/whole)
8. Metrics (density, velocity, tortuosity, dispersity, glomeruli)
9. Visualization (maps, panels, montages)
10. Python aggregation & correlations
11. Dashboard (Streamlit)

---

## Data model

- Blocks: \(IQ \in \mathbb{R}^{n_z \times n_x \times n_t}\)
- Maps: `MatOut`, `MatOut_vel`, `MatOut_zdir`
- Tables: RAW (voxel), TRACKS, SUMMARY (ROI)

---

## Key equations

### SVD filtering
\[
X = U\Sigma V^T, \quad X_{\text{filt}} = \sum_{k\in\mathcal{K}} \sigma_k u_k v_k^T
\]

### Velocity scaling
\[
v = \frac{\Delta s}{\Delta t}, \quad \lambda = \frac{c}{f}
\]

### Density
\[
\rho = \frac{N}{A}
\]

### Tortuosity
\[
T = \frac{L_{\text{path}}}{L_{\text{straight}}}
\]

### Dispersity
\[
\text{Dispersity} = 1 - R
\]

---

## Strengths
- End-to-end pipeline
- Hemodynamic separation (FAST/SLOW)
- Anatomical ROIs
- Cohort-ready outputs

## Limitations
- 2D plane
- SVD sensitivity
- tracking ambiguity in dense regions
