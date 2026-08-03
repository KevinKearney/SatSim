# Audit 0001: SatSim rendering-path differentiability

**ADR-0001 Action Item 5.** Read-only source audit of SatSim's rendering path to confirm or refute the
BLUF/Guiding-Constraint assumption that **Poisson noise sampling** and **annotation/truth-file logic**
are the *only* two gradient-breaking operations. **No patches were made** (Action Item 6 is gated behind
review of this audit and the deferred autodiff-framework decision).

- **Audited:** SatSim **0.25.2**, the installed source under
  `.../envs/satsim-env/Lib/site-packages/satsim/` (not the published docs).
- **Method:** static reading of the forward path **plus** empirical `tf.GradientTape` probes of the
  actual SatSim functions (results tabulated below). SatSim always runs TensorFlow **eagerly**
  (`configure_eager`), so gradient behavior is what a `GradientTape` sees.
- **Date:** 2026-08-03.

---

## Verdict (BLUF)

**The "only two exceptions" statement is too strong as literally written — it is refuted by at least one
additional hard break on the default forward path — but the ADR's underlying intent survives once the
differentiable objective is scoped properly.**

- ✅ **Poisson photon-noise sampling is confirmed** as a genuine, hard non-differentiability. TensorFlow
  registers **no gradient at all** for the Poisson sampler (`RandomPoissonV2`).
- ✅ **Annotation / truth-file logic is confirmed to sit outside the forward tensor path** — it is all
  post-hoc `.numpy()` extraction, exactly as the ADR assumes.
- ❌ **They are not the only two.** The **analog-to-digital conversion** (`analog_to_digital`) contains a
  `tf.math.floor` quantization that **kills the gradient** (empirically `grad = None`) and is squarely on
  the *default* forward path — it is a third hard break the ADR does not mention.
- ⚠️ **Two more, scope-dependent:** (a) the **default integer pixel deposition** (`add_counts`,
  `interpolation='floor'`) makes **sub-pixel position non-differentiable**, though SatSim already ships a
  **`bilinear` deposition mode that is position-differentiable**; (b) the geometry/photometry front-end
  (SGP4/Skyfield propagation, `mv_to_pe`/`pe_to_mv`) is **pure NumPy**, so nothing upstream of the
  rendered electron image carries a gradient — brightness/flux enters the graph as a constant.

The ADR's plan is **defensible if and only if** the differentiable objective is taken on the
**pre-A/D linear-electron image**, differentiated **with respect to brightness/flux (and PSF)**, not with
respect to sub-pixel position, and not through the A/D quantizer. That scoping is consistent with the
ADR's own "keep intermediate quantities in linear flux/count space" rule — but it needs to be stated,
because "only two exceptions" understates the work.

---

## The forward path (default `fftconv2p` mode)

Order of operations, from `satsim/satsim.py`:

| step | call | file:line | gradient |
|---|---|---|---|
| render objects+stars onto oversampled grid | `render_full(...)` | `satsim.py:1340`, `image/render.py:164` | TF-native (see notes) |
| deposit counts (scatter) | `add_counts` / `transform_and_*` | `image/fpa.py:261,350,423` | values ✅, position ❌ in `floor` mode |
| PSF convolution | `fftconv2p` | `image/render.py` | ✅ differentiable (FFT) |
| downsample oversampled→detector | `downsample` (conv2d/pool/sum) | `image/fpa.py:27` | ✅ |
| **photon (shot) noise** | `add_photon_noise` | `satsim.py:1463`, `image/noise.py:38` | ❌ **Poisson, no gradient** |
| read noise | `add_read_noise` | `satsim.py:1466`, `image/noise.py:64` | ✅ Gaussian reparameterization |
| **A/D conversion** | `analog_to_digital` | `satsim.py:1476`, `image/fpa.py:128` | ❌ **floor() quantization** |
| write FITS / annotations / truth | `.numpy()` + `write_frame` | `satsim.py:237,1528–1746` | — outside graph (by design) |

**Notes.** The default path (`sim.mode='fftconv2p'`, no `sim.render_size`) selects `render_mode='full'`
(`satsim.py:530`) → `render_full`, which stays in TensorFlow. The alternative `render_piecewise`
(selected only when `sim.render_size` is set, `satsim.py:542`) accumulates tiles into **NumPy buffers**
(`image/render.py:125–126`, `np.zeros(...)`), which **breaks the graph** — a conditional, opt-in break
for very large frames, not on the default path.

---

## Empirical gradient probes

`tf.GradientTape` over the real SatSim functions (16×16 test image / point sources):

| operation | probe | result | interpretation |
|---|---|---|---|
| `add_read_noise` | d(out)/d(input) | `grad mean=+1.0000, nonzero=256/256` | **differentiable** — Gaussian noise is additive/reparameterized |
| `add_photon_noise` | d(out)/d(rate) | `LookupError: gradient registry has no entry for RandomPoissonV2` | **hard break** — TF has *no* Poisson gradient |
| `analog_to_digital` | d(out)/d(input) | `grad = None` | **hard break** — `floor()` has zero/undefined gradient |
| `add_counts` (floor) | d(out)/d(**values**) | `grad = [1,1,1]` | brightness **differentiable** |
| `add_counts` (floor) | d(out)/d(**positions**) | `grad = None` | sub-pixel position **not** differentiable |
| `add_counts` (bilinear) | d(out)/d(**positions**) | `grad = [-3280, 13120, -54000]` | **differentiable** in the existing bilinear mode |

(Probe script preserved in the session scratchpad; it only *calls* SatSim functions inside a tape — no
source was modified.)

---

## The two ADR exceptions, confirmed

1. **Poisson photon-noise sampling** — `image/noise.py:38`
   (`tf.squeeze(tf.compat.v1.random.poisson(fpa, [1]))`; the multi-sample averaging branch at line 35 is
   the same sampler in a loop). TensorFlow registers no gradient for `RandomPoissonV2`; a tape over it
   raises `LookupError`. This is the canonical spot the ADR's "replace with a differentiable Gaussian
   reparameterization" targets. **Confirmed exactly as described.** (Note the ADR's proposed fix is
   already latent in the codebase: `add_read_noise` shows the reparameterized-Gaussian pattern that
   photon noise would adopt.)

2. **Annotation / truth-file logic** — the entire output stage is `.numpy()` extraction downstream of the
   forward tensors: `fpa_digital.numpy()` for the FITS write (`satsim.py:237`), segmentation
   (`satsim.py:1528,1558`), and every ground-truth plane (`satsim.py:1727–1746`,
   `target_pe/star_pe/background_pe/... .numpy()`), then `write_frame`/`write_annotation`
   (`io/satnet.py`). None of this feeds back into the rendered image. **Confirmed outside the forward
   path**, so excluding it from the graph (as the ADR plans) requires no change to the forward path — it
   is already severed.

---

## Additional gradient-breaking operations found (the refutation)

1. **A/D quantization — `analog_to_digital` (`image/fpa.py:105–140`).** `tf.math.floor(fpa_digital/gain)`
   (line 128) is a step function: gradient is zero a.e. and the tape returns `None`. The surrounding
   full-well/saturation clamps (`tf.where` at lines 125, 132, 138) additionally zero the gradient in
   clamped/saturated regions. This is a **third hard break, on the default path**, not covered by the
   ADR's two exceptions. *Mitigation when Action Item 6 happens:* differentiate the **pre-A/D linear
   electron image**, or use a straight-through estimator for the quantizer if DN-space gradients are ever
   required.

2. **Integer pixel deposition — `add_counts` default `interpolation='floor'` (`image/fpa.py:293–309`).**
   Positions are `tf.cast(..., tf.int32)` before `tensor_scatter_nd_add`, so gradient flows to the
   deposited **values (brightness)** but **not to sub-pixel position**. SatSim already provides a
   **differentiable alternative**, `_add_counts_bilinear` (`image/fpa.py:312–347`), which distributes each
   count over four neighbors with float weights `(1−dr)(1−dc)…` and is position-differentiable (probe
   above). *Implication:* if position/pointing is ever an optimization target, the render must use
   `bilinear` deposition throughout; brightness-only optimization is unaffected.

3. **NumPy front-end (geometry + magnitude→pe).** `mv_to_pe`/`pe_to_mv` (`image/fpa.py:143–191`) use
   `numpy` (`10**`, `np.log10`), and the astrometric propagation (SGP4/Skyfield in `satsim/geometry/*`) is
   NumPy/C. Object brightness therefore enters the graph as a **constant tensor**; there is no gradient
   path back to magnitude or orbit. This is **consistent with the ADR by design** (magnitude is
   human-facing/QC, never a differentiated node; geometry is not an optimization target), but it means the
   differentiable "brightness" input must be injected as a **live tensor from the future band-integration
   component**, not recovered from SatSim's `mv_to_pe`.

4. **Conditional NumPy buffer in `render_piecewise` (`image/render.py:125–126`).** Opt-in (only when
   `sim.render_size` is set); breaks the graph via `np.zeros` accumulators. Avoid `render_size` for any
   differentiable run, or port the stitch buffer to TF.

---

## Recommendations for Action Item 6 (patching) — *not to be done now*

Gated behind review of this audit and the framework decision. When it happens, the patch list implied by
this audit is:

1. **Replace the Poisson draw** (`add_photon_noise`) with a Gaussian reparameterization
   `signal + sqrt(signal)·ε`, `ε∼N(0,1)`, valid in the photon-count regimes in use — as the ADR states.
   (`add_read_noise` is already in this form and needs no change.)
2. **Take the differentiable objective on the pre-A/D linear-electron image**, i.e. leave
   `analog_to_digital` out of the graph, or add a straight-through estimator if DN-space gradients are
   needed. This is the item the ADR's "only two exceptions" misses.
3. **Use `bilinear` deposition** (`point_rendering`/`interpolation='bilinear'`) on any run where position
   is differentiated; keep `floor` only for brightness-only work.
4. **Feed brightness as a live tensor** from the band-integration component (Notebook 3 / Action Item 4),
   bypassing the NumPy `mv_to_pe`; never route a gradient through magnitude.
5. **Avoid `sim.render_size`** (piecewise NumPy buffers) in differentiable runs.
6. **Exclude annotation/truth/segmentation** — already severed; just do not add `.numpy()` hops into the
   forward path.

## Caveats / limits of this audit

- Scope was the **rendering + noise + A/D + output** path (`image/render.py`, `image/fpa.py`,
  `image/noise.py`, and the `satsim.py` driver around lines 1300–1780). The **ePSF renderer**
  (`render_epsf` / `image/epsf.py`) and cloud/background component builders were spot-checked (they use a
  `.numpy()` only for a static kernel-size lookup, `render.py:379`) but not exhaustively traced; a full
  Action-Item-6 pass should confirm no data-path `.numpy()` hides in `image/epsf.py`.
- "Differentiable" here means a `GradientTape` returns a finite gradient in eager mode; it does **not**
  assert numerical stability or that the gradient is useful for optimization (e.g. clamps give correct but
  zero subgradients in saturated regions).
- No claim is made about `tf.function`/graph-mode compilation; SatSim runs eager and this audit reflects
  that.
