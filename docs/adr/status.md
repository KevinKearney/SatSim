> **Superseded 2026-08-04.** This was a session-dated status report. Its content has been folded into `docs/architecture/current.md` (Architecture Rev 1), §§1-4 and §6, which is now the current status reference. Left in place as a historical record; no further entries will be appended here. Original content follows unchanged.

---

# R&D Status Report

**Session date:** 2026-08-03
**Branch:** `rnd/action-items-2-5` (baseline commit on `main`; nothing pushed)
**Scope:** unattended execution of the `prompt.md` queue (env hygiene + ADR-0001 Action Items 2, 3, 5,
and reconnaissance feeding Action Item 8). Companion to ADR-0001 and ADR-0002.

This is the single status report the operating rules asked for: what was completed, what is deliberately
not done, every assumption made, and the open decisions now waiting on Kevin.

---

## Completed this session

Each numbered item is its own commit on `rnd/action-items-2-5`.

| # | Item | Artifact | Commit |
|---|---|---|---|
| 0 | Pin `satsim-env` deps | `requirements.txt` (156→164 pins), `environment.yml` | `dc220ba` |
| 1 | Geometry/visualization notebook (AI2) | `notebooks/geometry_and_visualization.ipynb` | `fb9e3d5` |
| 2 | Band-integration validation notebook (AI3) | `notebooks/band_integration_validation.ipynb` | `cec020f` |
| 3 | Differentiability audit (AI5, read-only) | `docs/audits/0001-satsim-differentiability-audit.md` | `f0f99fe` |
| 4 | Injection-mechanism recon (feeds AI8) | `docs/recon/0001-injection-mechanism-reconnaissance.md` | `e658b23` |

All three notebooks execute top-to-bottom on the **Python (satsim-env)** kernel with **zero cell errors**
(verified via `nbconvert --execute`, then programmatic error/figure checks).

### Item 0 — dependency pinning
`requirements.txt` holds clean `name==version` pins from `pip list --format=freeze` (conda `file://` build
paths normalized out); `environment.yml` is a conda-forge py3.10 shell that pip-installs it. A full clean
rebuild was **not** run (multi-GB TensorFlow stack — not cheap, per the prompt's allowance); pins were
validated for internal consistency with `pip check`. Installed `pandas` (poliastro needed it, was missing).

### Item 1 — geometry & visualization (ADR AI2)
Verified SatSim's `save_czml` against source (`satsim/io/czml.py`) and a generated file: it writes valid
**CZML 1.0** (`satsim.czml`) — clock + observer (ground/space) with a time-varying rectangular sensor cone
+ per-target **inertial/ECI** position paths. **Findings vs ADR:** positions/paths/billboards/clock are
standard CZML (vanilla CesiumJS); the sensor cone uses the **AGI `agi_rectangularSensor` ion extension**
(plain Cesium ignores it). **SatViz/`satsimjs` documents a custom command schema, not a CZML loader** — so
SatSim-CZML → SatViz is *not* a drop-in path; vanilla CesiumJS is the reliable consumer. The Python-native
fallback (Skyfield + poliastro) runs here today and is the fastest geometry sanity check.

### Item 2 — band-integration validation (ADR AI3)
Built a synthetic (8,8,200) DIRSIG-shaped FLAM cube in a specutils `Spectrum1D`. **Resolved the ADR's open
question empirically:** `synphot.Observation` does **not** batch-integrate a multidimensional `Spectrum1D`
— a cube raises `ValueError: lookup_table should be an array with 1 dimensions`; only a 1-D `Spectrum1D` is
accepted, so **per-spaxel iteration is required** (root cause: `from_spectrum1d` → `Empirical1D`/`Tabular1D`
is 1-D only, `synphot/spectrum.py:927`). Validated SED×bandpass×QE → count rate and AB magnitude against a
hand-computed trapezoid reference: agreement to **2.9e-6** (count rate) and **6.6e-5 mag** (AB). This is the
reference the future differentiable component (AI4) must reproduce.

### Item 3 — differentiability audit (ADR AI5, read-only)
Read installed SatSim 0.25.2 source **and** ran `tf.GradientTape` probes on the real functions.
**Verdict: the ADR's "only two exceptions" is too strong as written.**
- ✅ **Poisson photon-noise sampling confirmed** as a hard break — TF registers **no gradient** for
  `RandomPoissonV2` (`image/noise.py:38`). `add_read_noise` is already a differentiable Gaussian
  reparameterization (`image/noise.py:64`).
- ✅ **Annotation/truth logic confirmed outside the forward path** — all post-hoc `.numpy()`
  (`satsim.py:237,1528–1746`).
- ❌ **A third hard break:** `analog_to_digital`'s `tf.math.floor` quantization kills the gradient
  (`image/fpa.py:128`; probe → `grad=None`), on the **default** path.
- ⚠️ Scope-dependent: default integer `floor` pixel scatter breaks **sub-pixel position** gradients
  (`fpa.py:293`) but a differentiable `bilinear` mode exists (`fpa.py:312`); `mv_to_pe`/SGP4/Skyfield
  front-end is NumPy (brightness enters as a constant); `render_piecewise` uses NumPy buffers (opt-in via
  `sim.render_size`). Full write-up + implied AI6 patch list in the audit doc. **No patches made.**

### Item 4 — injection reconnaissance (feeds ADR AI8)
Located the pre-noise seam: `satsim.py:1461` assembles
`fpa_conv = (fpa_conv_star + fpa_conv_targ + frame_bg_tf)*gain_tf + dc_tf`, then `add_photon_noise` draws
the **sole** shot noise on the sum. Ranked hook points (natural home = the per-frame `frame_bg_tf`
background term, which SatSim does **not** PSF-convolve — confirming the ADR's pre-blur requirement; plus
the existing but *static* `augment.background.stray` callable). Ruled out post-noise hooks. Surfaced the
integration constraints and open design questions. **No mechanism designed, chosen, or built.**

---

## Silent-failure catalogue (SatSim has no config schema validation)

Referenced by ADR-0002. Collected across Notebook 1 and this session; all reproduced.

1. **Unknown/misspelled keys are ignored.** A typo'd `sim` key (`spatal_osf`) runs with no error/effect.
2. **A target with no valid `mv`/`pe` renders with zero signal** (silently invisible; only hint is a
   `divide by zero in log10` warning).
3. **`stars.mv.bins` are N+1 edges, `density` are N counts** — mismatches silently zipped.
4. **Annotation coordinates are normalized (0–1), not pixels** — multiply by width/height before overlay.
5. **Illumination is not modeled unless `sim.enable_earth_shadow`** — targets render regardless of Earth
   shadow. (Item 1 cross-check: at the Notebook-1 epoch the ISS is **58° below the Maui horizon**, yet
   SatSim still produced a "detection." Geometry-check every scene.)
6. **Stars render but are not annotated** unless `sim.star_annotation_threshold` is set.
7. **`eager` argument is inert** ("has no effect. True only").
8. **Missing dependencies surface only as a bare `ModuleNotFoundError`** at `import satsim`.

---

## Assumptions made (per the "make a reasonable call and keep going" rule)

1. **Git layout.** The repo had **no commits**; I made a baseline commit on `main` (pre-existing
   scaffolding + completed Notebook 1) so the R&D branch had a sane base, then did all item work on
   `rnd/action-items-2-5`. Nothing pushed.
2. **Pins as `requirements.txt` + thin `environment.yml`.** The env is a hybrid conda+pip install; a raw
   `pip freeze` embeds non-portable `file://` paths, so I used `pip list --format=freeze`. Chose not to run
   a full clean rebuild (expensive) — validated with `pip check` instead.
3. **specutils constrained to 1.x.** Installed `specutils<2 astropy<6 asdf<3` → specutils 1.20.3, so astropy
   stayed 5.3.4 and SatSim's pins (`astropy<6`, `asdf<3`) were preserved. The latest specutils 2.x would
   have force-upgraded astropy≥6 and **broken SatSim**.
4. **synphot/astropy tension left as-is.** synphot 1.7.0 declares `astropy>=6` but runs correctly on 5.3.4
   for the band-integration used (verified). Did **not** upgrade astropy (would break SatSim). Documented in
   `requirements.txt` and flagged below.
5. **Canonical ISS TLE + epoch** (Vallado/`sgp4` example) reused across notebooks for a valid, reproducible
   target. Its epoch puts the target below the Maui horizon at the scene time — used deliberately as a
   teaching cross-check, and Skyfield's pass-finder locates a real visible pass for the sky-track plot.
6. **`satsimjs` not cloned/run.** The "no CZML loader" finding is from its published README/docs, not an
   exhaustive JS source audit (JS toolchain out of scope for a Python session). Flagged for confirmation
   when the front-end is stood up (AI9).
7. **Doc placement.** New `docs/audits/` and `docs/recon/` folders; final report at `docs/adr/status.md`
   (the path ADR-0002 already references).
8. **`.gitignore`** extended for generated artifacts (`notebooks/output/`, `*.fits`, `*.bsp`); the 17 MB
   `de421.bsp` and render outputs are not committed.

---

## Not done — deliberately (hard stops honored)

- **Autodiff framework choice (JAX/PyTorch/TF)** — not made. The differentiable band-integration component
  (AI4) was **not** started.
- **RSO-injection/light-curve mechanism** — investigated only (Item 4); **not designed or built**.
- **DIRSIG** — not installed, invoked, downloaded, or modified in any way.
- **`RSO_Sim` package / pydantic scaffolding** — none created (ADR-0002: waits for Notebook 4).
- **SatSim patches (AI6)** — none. The audit lists the implied patches; they stay gated behind your review
  and the framework decision.

Nothing was **blocked** in the failure sense — all five items completed. The dependency-resolution risk in
Item 2 (specutils vs astropy) was resolved by constraint, not abandoned.

---

## Open decisions now waiting on Kevin

1. **Autodiff framework (JAX vs PyTorch vs TensorFlow)** for the differentiable SatSim refactor + the
   band-integration reimplementation (ADR AI4). Deferred by the ADR; unblocks AI4 and AI6.
2. **RSO variation model:** brightness-only vs attitude/BRDF-driven glint across a frame sequence. This
   drives the injection seam choice (static `augment` hook vs per-frame source seam) and how heavily the
   band-integration layer is exercised per frame. (Recon doc §Open questions.)
3. **AI6 scope in light of the audit:** the ADR's patch list should grow from two items to at least three —
   confirm that the differentiable objective is taken on the **pre-A/D linear-electron image** (leaving
   `analog_to_digital`'s `floor` out of the graph), and standardize on `bilinear` deposition if position is
   ever differentiated. Please review the audit before any patching.
4. **Long-term astropy/synphot/specutils resolution:** the env currently satisfies SatSim (astropy<6) at the
   cost of a declared-but-not-enforced synphot floor. Fine for R&D; revisit if synphot/specutils are carried
   past the R&D phase (the ADR retires synphot anyway).
5. **Visualization front-end (AI9):** confirm from `satsimjs` source whether its viewer exposes the
   underlying `Cesium.Viewer` (so a `CzmlDataSource` can be added) or whether a CZML→command-schema
   converter is needed. Until then, vanilla CesiumJS is the baseline and Skyfield is the day-to-day check.

---

## Environment changes made this session
- Installed into `satsim-env`: `pandas` (poliastro dep), `specutils==1.20.3` (+ gwcs/ndcube/asdf-astropy,
  constrained to preserve astropy 5.3.4). All captured in `requirements.txt`.
- (Prior session, for context) fixed the `satsim-env` Jupyter kernelspec to point at the env interpreter;
  installed SatSim's missing runtime deps + `nbformat`/`nbconvert`.

## Note for future runs
- Use `python -m nbconvert` rather than the `jupyter` launcher shim — the latter hit a transient
  "Access is denied" on Windows this session.
