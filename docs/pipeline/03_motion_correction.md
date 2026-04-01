# 3. Motion Correction (SRUS)

## Purpose

Motion correction is an optional but very important stabilization step in the refactor. Its role is to remove tissue or probe motion before microbubble localization and tracking are performed.

Relevant files include:

- `0_config/src/srus/run_motion_correction_srus.m`
- `0_config/src/blocks/build_blocks_with_srus_mc.m`
- output `result/Motion_corrected_data.mat`

---

## Why motion correction matters in ULM

ULM assumes that the motion being reconstructed is primarily **microbubble flow inside vessels**. In practice, the raw cine may also contain motion due to:

- breathing
- kidney displacement
- hand or probe movement
- slow drift of the imaging plane

If that motion is not corrected, the pipeline may mistake tissue displacement for vascular structure. This can lead to:

- broadened vessels
- artificial trajectories
- wrong direction maps
- biased velocities
- misleading density maps

In short, motion correction protects the physical meaning of the later tracking stage.

---

## Conceptual model

Let \(I_t(x,z)\) be the image at time \(t\). If the tissue is moving under a transformation \(T_t\), then the observed frame can be described conceptually as:

\[
I_t^{\text{obs}}(x,z) = I_t^{\text{true}}(T_t(x,z))
\]

Motion correction tries to estimate \(T_t\) and apply the inverse transform so that the anatomy is aligned over time:

\[
I_t^{\text{aligned}}(x,z) = I_t^{\text{obs}}(T_t^{-1}(x,z))
\]

After this alignment, remaining moving bright structures are more likely to correspond to contrast-agent dynamics rather than organ drift.

---

## How it is integrated in the refactor

Your batch driver asks the user to choose a global cropping mode for the run. One branch is a motion-corrected path using SRUS, and the other is a simpler ROI crop without MC.

This is a good architectural choice because it means the same refactor can support both:

- fast exploratory runs without MC
- more stable runs with motion correction

without duplicating the whole pipeline.

---

## Output of this stage

The SRUS branch produces motion-corrected data typically saved as:

```text
result/Motion_corrected_data.mat
```

That corrected data are then used during block construction or subsequent ULM steps.

---

## Why this improves downstream tracking

Tracking relies on linking positions of the same microbubble across frames. If the whole tissue background is drifting, a valid vascular path and an invalid tissue-induced apparent path can become difficult to distinguish.

After motion correction:

- vessel geometry is sharper
- track linking is more coherent
- ROI overlays stay aligned
- velocity maps become more interpretable

---

## Limitations

Motion correction is helpful, but it is not magic. It can still fail if:

- contrast is too weak
- the motion is too large or non-rigid
- registration locks onto the wrong structure
- the imaging plane itself changes substantially

That is why the pipeline still benefits from later QC of maps and trajectories.

---

## Inputs and outputs

### Inputs
- raw cine or pre-block image sequence
- reference frame or registration model

### Outputs
- motion-corrected data
- corrected blocks for PALA processing

---

## Summary

This stage protects the meaning of microbubble flow reconstruction by separating **true intravascular motion** from **global image motion**. In a translational renal ULM workflow, that separation is essential for trustworthy density, velocity, and tortuosity estimates.
