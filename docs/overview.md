# Overview

## What is ULM?

Ultrasound Localization Microscopy (ULM) achieves **super-resolution** by tracking individual microbubbles beyond the diffraction limit.

### Diffraction limit

\[
\lambda = \frac{c}{f}
\]

with \(c \approx 1540\,\text{m/s}\).

Conventional imaging is PSF-limited. ULM instead estimates **subpixel positions** of sparse emitters.

---

## Signal model

A bubble is approximated as:

\[
I(x,z,t) = \delta(x-x_0, z-z_0) * \text{PSF}(x,z) + \eta
\]

Localization estimates \((x_0, z_0)\) from the blurred response.

---

## From images to flow

1. **Filter clutter** (SVD)
2. **Detect peaks** (localization)
3. **Link over time** (tracking)
4. **Infer kinematics**

\[
v = \frac{\Delta s}{\Delta t}
\]

---

## Why kidneys?

Renal compartments (cortex/medulla) have distinct microvascular organization. ULM enables:

- perfusion-like assessment
- structural complexity (tortuosity)
- directional organization (dispersity)
