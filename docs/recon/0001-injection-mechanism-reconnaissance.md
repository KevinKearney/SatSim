# Recon 0001: SatSim background-injection hook points

**Feeds ADR-0001 Action Item 8 (pulled forward per prompt) / ADR-0002 Notebook 4.**
**Investigation only.** This note identifies *where* an externally-supplied background image (e.g. a
DIRSIG-derived, band-integrated background) could be injected into SatSim's pipeline before its shot-noise
draw, and surfaces the open design questions. **No injection mechanism is designed, chosen, or built** —
that design is a hard-stop decision reserved for Kevin.

- **Source read:** SatSim **0.25.2** installed source (`satsim/satsim.py`, `satsim/image/*`,
  `satsim/background/*`). Read-only.
- **Date:** 2026-08-03.

---

## What the ADR requires of an injection point

From ADR-0001 §Decision.3: SatSim injects the RSO onto the DIRSIG background via a mechanism (TBD) where
"the injected background must be **noiseless** and **pre-convolved with the system PSF** before injection,
so that SatSim's own shot-noise draw remains the **sole noise source** and background/target noise is
never double-counted."

So a valid hook must be **(a) before** `add_photon_noise`, **(b)** in the same electron units / pipeline
stage as the rendered signal, and **(c)** fed a background that the caller has already PSF-blurred and
left noiseless.

## The pipeline seam

The per-frame electron image is assembled and noised in `satsim/satsim.py`:

```python
# satsim.py:1461  -- electron image assembled from all sources
fpa_conv = (fpa_conv_star + fpa_conv_targ + frame_bg_tf) * gain_tf + dc_tf
# satsim.py:1462-1463  -- the SOLE shot-noise draw, on the summed signal
if ssp['sim']['enable_shot_noise'] is True:
    fpa_conv_noise = add_photon_noise(fpa_conv, ssp['sim']['num_shot_noise_samples'])   # Poisson
# satsim.py:1466  -- read noise, then A/D
fpa, rn_gt = add_read_noise(fpa_conv_noise, rn, en)
fpa_digital = analog_to_digital(fpa + bias_tf, ...)   # satsim.py:1476
```

Key structural facts established by reading the surrounding code:

- **`fpa_conv_star` / `fpa_conv_targ`** come out of `_render(...)` (`satsim.py:1441`), which **PSF-convolves**
  the star/target scene. **`frame_bg_tf` does NOT pass through any PSF convolution** — it is added directly
  to the already-convolved signal. → This concretely confirms the ADR: **SatSim will not blur an injected
  background for you; it must be pre-blurred.**
- **`frame_bg_tf`** (`satsim.py:1450`) is `frame_pre_cloud_bg_tf`, a **per-frame** sum of background-component
  tensors (`satsim.py:1191`), each `tf.cast(..., tf.float32)`. It is broadcast-added to the detector-space
  `fpa_conv_star`+`fpa_conv_targ`, so it may be a scalar **or a full detector-shaped 2-D tensor** — a
  detector-shaped external array drops in without shape trouble.
- Being inside the `frame_num` loop, `frame_bg_tf` is recomputed **every frame** → a **time-varying**
  background is natural at this seam.
- The Poisson draw is on `(signal + bg) * gain + dark`. An injected background must therefore be in the
  **same pre-gain electron unit and detector-space resolution** as `fpa_conv_star` (i.e. added into the
  parenthesized term, not after `* gain_tf`).

## Candidate hook points (ranked; not a recommendation)

| # | hook | file:line | pre-noise? | per-frame? | notes |
|---|---|---|---|---|---|
| 1 | **the `frame_bg_tf` background term** | `satsim.py:1450`, summed at `:1461` | ✅ | ✅ | The natural architectural home. Background is already an additive, pre-noise, detector-space term. Not PSF-convolved by SatSim → caller pre-blurs. Would need a code seam or a per-frame background-component provider. |
| 2 | **`augment.background.stray` callable** | `satsim.py:1000–1004` | ✅ | ❌ (applied **once**, at init) | *Existing config hook.* `bg = ssp['augment']['background']['stray'](bg)`, NaN-guarded, folded via `apply_background_stray_augmentation` into `background_components` → flows into `frame_bg_tf`. Accepts a 2-D array. **Limitation:** runs a single time before the frame loop, so it gives a *static* injected background; time variation is not expressed here without extension. |
| 3 | direct add into the `fpa_conv` sum | `satsim.py:1461` | ✅ | ✅ | Equivalent to #1 but as an explicit `+ external_bg_tf` term; requires a source-level seam. |

**Explicitly NOT valid for background injection (they sit *after* the shot-noise draw, violating the ADR's
"sole noise source" rule):**

- `augment.image.post` (`satsim.py:1479–1485`) — operates on `fpa_digital`, i.e. **post-A/D**. A background
  added here receives neither shot noise nor correct quantization interaction.
- `misc` noise (`satsim.py:1467–1472`) — added **after** read noise.

There is also an `augment.fpa.psf` hook (`satsim.py:1300`) that replaces the PSF tensor — relevant to the
"which PSF do we pre-blur with?" question below, not an injection point itself.

## Integration constraints surfaced (facts, not decisions)

1. **PSF pre-blur is mandatory and on us.** The background path skips convolution; the injected image must be
   convolved with **the same PSF SatSim uses** (poppy/Gaussian per config, optionally overridden by
   `augment.fpa.psf`). Sourcing/҆matching that exact PSF is an open item.
2. **Unit & stage matching.** Inject into the pre-gain, detector-space electron term (`satsim.py:1461`
   parenthesis), in `pe`, at detector resolution (not oversampled).
3. **`enable_shot_noise` must be `True`** and the injected background must be **noiseless**, or the
   sole-noise-source invariant breaks.
4. **Registration.** The DIRSIG background frame must be pixel-registered to SatSim's detector grid and
   pointing/track for the frame's epoch — SatSim owns geometry (site/track), the background owns its own
   scene; aligning them is unowned glue (ADR "Harder" consequence).
5. **Per-band.** The injected background is one image **per bandpass**, produced by the band-integration
   layer (Notebook 3) from the DIRSIG cube — it must use the *same* bandpass×QE as the target's magnitude,
   or the color handoff is inconsistent.

## Open design questions (the deliverable — for Kevin to decide)

These are surfaced, not answered. Several are explicit hard stops.

1. **What varies across a frame sequence?** (ADR's own open question.) Brightness-only (a per-frame scalar
   on the target's `pe`, cheap — feeds `pe_obs` per frame), or **attitude/BRDF-driven glint** (a per-frame
   *spectral* change, which must run through the band-integration layer each frame, far more expensive)?
   This choice drives everything downstream and is **Kevin's to make** (session hard stop).
2. **Static vs. per-frame injection seam.** Hook #2 (`augment.background.stray`) exists today but is
   applied once; hooks #1/#3 are per-frame but need a code seam. Does the light-curve/background variation
   require per-frame backgrounds (→ #1/#3), or is a static background with only the *target* varying
   sufficient (→ #2 usable now)?
3. **Adapter vs. fork.** ADR-0002 and ADR-0001 both prefer *not* patching SatSim. Can injection live behind
   the existing `augment` callable surface (config-level, no fork), or does per-frame time variation force a
   source seam at `:1461`? If a seam is needed, is it an upstream-able contribution or an RSO_Sim-owned patch?
4. **PSF single-source-of-truth.** Where does the pre-blur PSF come from so it provably matches SatSim's
   render PSF (poppy config, sampling, oversampling)? Does `augment.fpa.psf` need to be the shared origin?
5. **Registration/time-indexing ownership.** Who aligns the DIRSIG background (its scene, cadence) to
   SatSim's detector grid and track epoch per frame — and in which coordinate frame?
6. **Interaction with the band-integration layer.** The background and the target must share the same
   bandpass×QE and live in linear `pe` (not magnitude) at injection; confirm the handoff format (per-band
   `pe` map) end-to-end from Notebook 3 → this seam.

## Boundaries honored

- No injection mechanism designed, prototyped, or chosen.
- No `RSO_Sim`/pydantic scaffolding (ADR-0002: waits for Notebook 4, after the injection design is settled).
- DIRSIG not invoked or touched.
- The light-curve/attitude-variation design decision is left entirely to Kevin.
