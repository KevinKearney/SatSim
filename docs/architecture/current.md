# RSO_Sim Architecture — Rev 2

**Status:** living document — edited in place going forward; cite by revision number, not by re-reading history.
**Date:** 2026-08-04
**Owner:** Kevin

This document is the current-state description of the RSO_Sim architecture: what's decided, what's built, what's open. It supersedes `docs/adr/0002-notebook-sequence-and-pydantic-timing.md` as the roadmap reference and `docs/adr/status.md` as the status reference (both left in place, marked superseded, for history). Decision provenance — why, and what alternatives were rejected — lives in the immutable record: `docs/adr/0001-hybrid-synphot-satsim-pipeline.md` (the architecture decision itself), `docs/audits/0001-satsim-differentiability-audit.md`, and `docs/recon/0001-injection-mechanism-reconnaissance.md` (investigation records). This document cites that reasoning; it does not duplicate it.

## Changelog

| Rev | Date | Change | Basis |
|---|---|---|---|
| 1 | 2026-08-04 | Initial living doc. Synthesizes ADR-0001, ADR-0002 (retired), the differentiability audit, the injection recon, and status.md into one current-state document. | ADR-0001, ADR-0002, Audit-0001, Recon-0001, status.md |
| 2 | 2026-08-04 | §4: split the illumination/eclipse gap from the horizon/occlusion gap — they were conflated as one item, but `enable_earth_shadow` (optional, off by default) addresses only illumination and has no bearing on horizon occlusion, which has no check at all anywhere in SatSim's geometry path. | Source read of `satsim.py`/`geometry/shadow.py`; `notebooks/geometry_and_visualization.ipynb` |

## 1. Pipeline architecture (current)

Three tools, separation of concerns, no tool asked to do another's job (ADR-0001 §Decision):

- **synphot** (R&D-phase only) — SED × bandpass × QE → magnitude, operating on DIRSIG hypercubes held in a specutils `Spectrum1D` container. Retired once the differentiable band-integration component (below) exists.
- **SatSim** — image formation + annotation. Consumes literal magnitudes; owns geometry, PSF, detector noise chain, truth-file generation. Treats brightness as a scalar — no native color term.
- **DIRSIG** — offline radiometric background/target truth (noiseless, precomputed). Not yet invoked in this project.
- **Visualization** — Skyfield + poliastro is the working path today, not a fallback. SatSim's CZML export (`save_czml`) writes valid CZML 1.0 consumable by vanilla CesiumJS, but SatViz/`satsimjs` uses a custom command schema, not a CZML loader — no drop-in integration exists. Whether `satsimjs` exposes an underlying `Cesium.Viewer` that a `CzmlDataSource` could attach to is still open (§5, §6 item 4).

## 2. Differentiability status

Audit-0001 (read-only source audit + `tf.GradientTape` probes on SatSim 0.25.2) found more than ADR-0001 originally assumed. Confirmed hard breaks on SatSim's default forward path:

1. Poisson photon-noise sampling (`add_photon_noise`) — TensorFlow registers no gradient for `RandomPoissonV2`.
2. A/D quantization (`analog_to_digital`'s `tf.math.floor`) — gradient is `None`. Not in the ADR's original two-exception list.

Confirmed outside the forward path by design, no change needed: annotation/truth-file logic (all post-hoc `.numpy()` extraction).

Scope-dependent, not yet a decision:
- Default integer (`floor`) pixel deposition breaks sub-pixel position gradients; a differentiable `bilinear` deposition mode already exists in SatSim.
- The geometry/magnitude front end (SGP4/Skyfield, `mv_to_pe`) is pure NumPy — brightness enters the graph as a constant. Consistent with the ADR (magnitude is human-facing/QC, never differentiated), but it means the future band-integration component must feed brightness in as a live tensor, not recover it from SatSim's own conversion.
- `render_piecewise` (opt-in via `sim.render_size`) uses NumPy accumulator buffers, breaking the graph — avoid for differentiable runs.

Implied patch list for the eventual patching work — six items, not the ADR's original two (full detail in Audit-0001 §Recommendations): replace the Poisson draw with a Gaussian reparameterization; take the differentiable objective on the pre-A/D linear-electron image; standardize on `bilinear` deposition wherever position is differentiated; feed brightness as a live tensor from the band-integration component; avoid `sim.render_size` in differentiable runs; keep annotation/truth extraction outside the graph (already true — no action needed).

**Not started:** the autodiff framework choice, the differentiable band-integration component, and any patching of SatSim itself. All gated on the framework decision (§6).

## 3. Injection mechanism status

Recon-0001 (read-only source read) located the pipeline seam and ranked candidate hooks; no mechanism is designed or built.

The per-frame electron image is assembled at `satsim.py:1461`: `fpa_conv = (fpa_conv_star + fpa_conv_targ + frame_bg_tf) * gain_tf + dc_tf`, followed immediately by the sole shot-noise draw. `frame_bg_tf` is not PSF-convolved by SatSim — confirming the ADR's requirement that an injected background must be pre-blurred by the caller.

Ranked hooks: (1) the `frame_bg_tf` term itself — per-frame, pre-noise, but needs a code seam; (2) the existing `augment.background.stray` config callable — already accepts a 2-D array with no source changes required, but runs once at init, so it only supports a static background; (3) a direct additive term at the same line as (1), functionally equivalent to (1).

Integration constraints established (facts, not decisions): PSF pre-blur is mandatory and on us; injection must land in the pre-gain, detector-space electron term, in `pe`, at detector resolution; `enable_shot_noise` must stay `True` and the injected background must be noiseless; DIRSIG-to-SatSim registration (pixel grid, pointing, epoch) is unowned glue; the injected background must share the same bandpass×QE as the target's magnitude.

**Blocking design question:** what varies across a frame sequence — brightness-only (cheap, a per-frame scalar) or attitude/BRDF-driven glint (expensive, runs through the band-integration layer every frame)? This choice determines whether hook (2) is sufficient or a source seam at hook (1)/(3) is required, and is reserved for Kevin (§6).

## 4. Known config-validation gaps (SatSim has no schema validation)

Collected across the tutorial notebook and the unattended session; all reproduced empirically. These are requirements for the eventual RSO_Sim config wrapper (§5), not standalone patches to build now:

1. Unknown/misspelled config keys are silently ignored (e.g. `spatal_osf`).
2. A target with no valid `mv`/`pe` renders with zero signal, silently — only symptom is an unrelated `divide by zero in log10` warning.
3. `stars.mv.bins` (N+1 edges) vs. `density` (N counts) length mismatches are silently zipped.
4. Annotation coordinates are normalized [0,1], not pixel — undocumented in-band, must scale by width/height.
5. **Illumination/eclipse** is not modeled unless `sim.enable_earth_shadow` is explicitly set — off by default, targets render as fully lit regardless of Earth's umbra. Source: `earth_shadow_umbra_mask` (`satsim.py:2495`, `geometry/shadow.py`), a Sun–Earth–target geometry check gated behind this one config flag.
6. **Horizon/occlusion has no check at all**, with or without any config flag. SatSim's geometry path contains exactly two gates — an angular FOV filter (`_batch_sgp4_fov_filter`, is the target's line-of-sight within the sensor's field-of-view cone) and the illumination check in item 5 above — and neither tests whether the observer's line of sight to the target is physically blocked by the Earth. Confirmed: the ISS at 58° below the Haleakalā horizon rendered as a "detection" in the visualization notebook, fully lit, correctly inside the FOV, and geometrically impossible. This is a distinct, more serious gap than item 5 — it is not addressable by `enable_earth_shadow` or any other existing flag; it requires a check SatSim has no equivalent of anywhere in its source.
7. Stars render but go unannotated unless `sim.star_annotation_threshold` is set.
8. The `eager` config argument is inert.
9. Missing dependencies surface only as a bare `ModuleNotFoundError` at import time.

## 5. Roadmap

`RSO_Sim` is the working name for the eventual integration package spanning synphot, SatSim, and DIRSIG. Reserved, not scaffolded — no defined API exists yet.

Notebook sequence and status:

1. Using SatSim — complete.
2. Geometry/visualization — complete (§1; the `satsimjs` viewer-capability question in §6 item 4 remains open within it).
3. Band-integration validation — complete. `synphot.Observation` does not batch-integrate a multidimensional specutils `Spectrum1D` (raises `ValueError`; per-spaxel iteration required — root cause: `from_spectrum1d`'s `Empirical1D`/`Tabular1D` path is 1-D only). Physics validated to 2.9e-6 (count rate) / 6.6e-5 mag (AB) against a hand-computed reference.
4. Injection/light-curve prototype — reconnaissance complete (§3); design blocked on the RSO variation model decision (§6).
5. Differentiability audit — complete (§2).

Rule: do not extract RSO_Sim classes off Notebooks 1–3 individually — the class boundary only becomes visible once two tools' config/data actually have to reconcile, which first happens at Notebook 4.

Pydantic: introduce at Notebook 4, not before — its role is the schema-validation layer SatSim structurally lacks (§4's catalogue is its requirements list, including the horizon gate in item 6). Config-only, never wrapping tensor data — DIRSIG hypercubes and SatSim's rendered arrays must stay plain numpy/TF/JAX for the eventual autodiff graph. Implemented as an adapter (`to_satsim_config()`-style export), not a patch to SatSim's or DIRSIG's own config handling. Pair with `astropy.units`/`pint` for magnitude/wavelength/flux fields — pydantic alone doesn't validate physical units.

Visualization front-end: confirm from `satsimjs` source whether its viewer exposes the underlying `Cesium.Viewer` (so a `CzmlDataSource` could be attached) or whether a schema converter would be needed — the "no CZML loader" finding above is from `satsimjs`'s published docs, not an exhaustive JS source audit. Until resolved, Skyfield/poliastro is the working geometry check and vanilla CesiumJS (not SatViz) is the baseline CZML consumer.

## 6. Open decisions

1. **Autodiff framework** (JAX / PyTorch / TensorFlow). Unblocks the differentiable band-integration component and any SatSim patching (§2).
2. **RSO variation model** — brightness-only vs. attitude/BRDF-driven glint. Determines the injection-seam design (§3).
3. **Patch-list confirmation** for the eventual SatSim patching work — review Audit-0001's six-item list (§2) before it's built; in particular, confirm taking the differentiable objective on the pre-A/D linear-electron image.
4. **`satsimjs` viewer capability** — confirm or refute the `Cesium.Viewer` exposure question (§5) against actual JS source, not just docs.

Background, not requiring near-term action: the `astropy`/`synphot`/`specutils` version tension in `satsim-env` (astropy held at 5.3.4 for SatSim; synphot's declared `≥6` floor is unenforced and currently works) — acceptable for R&D, revisit only if synphot/specutils carry past this phase, which the ADR doesn't plan for anyway.

## 7. Environment

`satsim-env`: pinned in `requirements.txt` (164 packages) + `environment.yml`. TensorFlow is CPU-only on native Windows — GPU requires WSL2; unresolved, relevant before any differentiable-patch work (§2) that needs to run at scale. `specutils` constrained to `<2` (→ 1.20.3) to hold `astropy<6`, preserving SatSim's pin; see §6 background item.

## 8. Superseded documents

- `docs/adr/0002-notebook-sequence-and-pydantic-timing.md` — was filed as an ADR but was always a roadmap, not a point-in-time decision with alternatives considered. Content folded into §5 above. Left in place for git history; the ADR number `0002` is retired, not reused.
- `docs/adr/status.md` — session-dated status report (2026-08-03). Content folded into §§1–4, 6. Left in place as a historical record; no further entries — status going forward is tracked in this document's Changelog and body. Note: its original text also conflates the illumination and horizon gaps (see Rev 2 changelog above) — left as-is, since the point of freezing it was to preserve what was actually reported at the time, not to keep it accurate going forward.
