# ULM Refactor Pipeline

## End-to-end Ultrasound Localization Microscopy for Renal Microvasculature

This project implements a **modular, reproducible, and clinically oriented ULM pipeline** that transforms raw ultrasound cine data into **quantitative biomarkers** of renal microvascular structure and flow.

### Core capabilities
- Super-resolution reconstruction via microbubble localization & tracking
- Dual-regime analysis (**FAST vs SLOW**) for hemodynamic heterogeneity
- ROI-based quantification (cortex, medulla, whole kidney)
- Multi-scale outputs: voxel, track, ROI, cohort
- Python-based aggregation, correlations, and dashboard

---

## From data to biomarkers

```text
DICOM cine
→ blocks (IQ)
→ motion correction (optional)
→ SVD + temporal filtering
→ localization → tracking
→ MatOut (structure, direction, velocity)
→ FAST / SLOW split
→ ROI metrics (density, velocity, tortuosity, dispersity, glomeruli)
→ cohort aggregation (Python)
→ dashboard (Streamlit)
```

---

## What you get

- **Density (counts/mm²)**: vascular occupancy surrogate  
- **Velocity (mm/s-like)**: flow dynamics surrogate  
- **Tortuosity**: path curvature complexity  
- **Dispersity**: directional disorder  
- **Glomeruli metrics**: localized microvascular features

---

## Documentation map
- **Overview**: physics & intuition of ULM  
- **Architecture**: how modules interact  
- **Code Map**: exact mapping to scripts  
- **Pipeline/**: step-by-step implementation
