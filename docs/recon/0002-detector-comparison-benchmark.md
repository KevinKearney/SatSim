# Recon 0002: Detector-comparison radiometric benchmark

**Investigation record.** Companion to `notebooks/radiometric_detector_comparison.ipynb`. This is a
benchmark/analysis note, **not** an architecture decision — it introduces no design and changes nothing in
the pipeline. Cite `docs/architecture/current.md` for architecture; this note only records what the
notebook implements, every non-paper value it assumes, and the open questions found while building it.

- **Built:** `notebooks/radiometric_detector_comparison.ipynb` (runs on `Python (satsim-env)`, 0 cell
  errors, verified for all three `ACTIVE_DETECTOR` values).
- **Source model:** Waldmann et al. 2025 (AMOS), Appendix A radiometric model (eqs 1–11), as summarized
  in `prompt-detector-comparison.md`. The paper itself was not available; the equation *forms* are a
  parametrization of that summary, not the paper verbatim.
- **Date:** 2026-08-04.

---

## 1. What the notebook does

Implements a wavelength-resolved photon-rate + SNR chain and swaps between three detectors
(**Ninox 1280**, **Teledyne Judson SCION VisGaAs**, **toy_model**) under one fixed optical/target/
atmosphere scenario, to isolate each detector's contribution to integrated SNR.

Chain (eq → implementation):

- **Eq 1** — uniform 1 nm grid (`λ_ref=1 µm`, `R=1000`), spanning 0.4–1.7 µm (union of all detectors' ranges).
- **Eq 2** — solar irradiance as a 5778 K blackbody normalized to 1361 W/m² (WHI proxy).
- **Eqs 3–7** — four-term BRDF (diffuse body+panel, gated specular glint, and their Earthshine
  counterparts) → irradiance at the aperture, for a placeholder 1U-CubeSat-LEO-500 km target.
- **Atmosphere** — parametric MODTRAN substitute: λ⁻⁴ Rayleigh sky + flat transmission windows.
- **Eqs 8–9** — aperture/throughput from the paper's Table 1 (0.7 m, f/6.5, 4540 mm, 3-surface Al) →
  collected power → photon rate → electron rate, band-limited by the active detector's QE(λ).
- **Eqs 10–11** — shot + dark + sky + read noise budget → integrated SNR, with a per-wavelength
  noise-budget plot and a three-detector comparison table.

Detector parameters are read from `configs/detector_configs.json` (never hardcoded).

## 2. Every assumed / placeholder value introduced

None of these come from the paper (which publishes only class names, Table 1 optics, and one sky-ratio
benchmark). All are flagged inline in the notebook and reproduced here so the provenance is auditable.

| quantity | value used | nature |
|---|---|---|
| Solar spectrum | 5778 K blackbody @ 1361 W/m² | **substitute** for the paper's WHI curve (no Fraunhofer lines / true UV-IR shape) |
| Target class | 1U CubeSat, LEO, 500 km slant range | assumed representative "conservative" case (paper names class only) |
| Body / panel area | 0.01 m² / 0.06 m² | **placeholder** |
| Body / panel albedo | 0.20 / 0.25 | **placeholder** |
| Earth albedo (Earthshine) | 0.30; sunlit-Earth solid-angle frac 0.30 | **placeholder** |
| Diffuse phase-function value | 0.50 | **placeholder** |
| Specular lobe / alignment | 0.05 sr / alignment = 0 (glint OFF) | conservative sustained geometry; glint modeled but disabled |
| BRDF functional forms | Lambertian diffuse + gated specular lobe + Earthshine | **parametrization** of eqs 3–7, not verbatim |
| Atmosphere transmission | flat windows 1.0/1.2/1.6 µm + visible, 0.25 floor | **illustrative**, no absorption-line structure |
| Sky radiance shape | λ⁻⁴ Rayleigh; **Mie = 0, airglow = 0** | omissions explicitly flagged, not folded into Rayleigh |
| Daytime sky absolute level | `SKY_VIS_REF = 20` W/m²/sr/µm (visible zenith) | **placeholder** absolute level; drives the §11 regime finding |
| Mirror throughput | 0.98³ ≈ 0.94 (3 surfaces) | **assumed** (paper gives no coefficient) |
| Aperture obscuration | none (unobstructed) | **assumed** (secondary not published) |
| Integration time / PSF footprint | 1 ms / 5 px | **placeholders**, held fixed across detectors |
| QE(λ), ninox & scion | flat `qe_peak` within `spectral_range_um` | **assumed** — `qe_curve` is null for *all three* detectors |
| QE(λ), toy_model | `TOY_MODEL_QE_PLACEHOLDER = 1.0` (flat) | **NON-PHYSICAL** placeholder; toy_model SNR is not comparable |
| toy_model pixel pitch | 10 µm | **placeholder** (config has none; used only for sky solid angle) |

Representative outputs (with the above placeholders, 1 ms, daytime sky): SNR_int ≈ Ninox 28 / SCION 23 /
toy 25; night (low sky): Ninox 61 / SCION 80 / toy 101. Absolute values are scenario-dependent to within
the placeholders; the **relative** behavior is the point.

## 3. Config gaps handled (per the prompt's policy, not papered over)

- **toy_model has no QE at all** (`qe_peak`, `qe_curve`, `spectral_range_um` all null). Handled with the
  single named `TOY_MODEL_QE_PLACEHOLDER = 1.0`, a loud runtime warning, and a "QE ASSUMED — NOT PHYSICAL"
  label on its row. No plausible-looking QE was invented.
- **scion_visgaas noise numbers are upper bounds** ("< 30 e−", "Target < 15 e−/px/s"), not measurements.
  Used as given, labelled as bounds in the output.

## 4. Open question — the Ninox ADU-vs-electron discrepancy (unresolved, by instruction)

`configs/detector_configs.json` carries two disjoint Ninox noise sets that do not reconcile:

| domain | read noise | dark current | source |
|---|---|---|---|
| paper (ADU/cts) | 224.1 ADU (high gain) | 205.7 cts/s | Waldmann et al. 2025, Table 3 |
| datasheet (e−, −15 °C) | 35 e− (hg typical) | 2000 e−/s (typical) | Raptor Ninox 1280 datasheet |

A single gain cannot map both: the gain that takes 224.1 ADU → 35 e− (≈ 0.156 e−/ADU) takes 205.7 cts/s →
~32 e−/s dark, i.e. **~50× below** the datasheet's 2000 e−/s typical. Possible explanations (not
resolved here): the paper's ADU numbers were taken at a different operating temperature / gain / exposure
than the datasheet's −15 °C typicals; or one of the two is mislabelled. Because eq 10 is defined in
photo-electrons, the notebook uses the **datasheet electron values** and does **not** derive a gain to
force a fit. Flagged as open; resolving it needs the paper's actual gain/temperature operating point.

## 5. Additional finding — the atmosphere is load-bearing at daytime sky levels

The prompt's premise is that the atmosphere's absolute level is *not* load-bearing for a detector-only
comparison (only its consistency across runs matters). That holds **only while sky background is a
sub-dominant noise term**. At the realistic daytime sky level used here, the budget is sky-dominated and
the detector ranking **flips**:

- **Night (sky ≈ 0):** signal/read-limited → SCION wins (SNR 80) on its wider 0.4–1.7 µm band and lower
  read noise; Ninox 61.
- **Day (placeholder sky):** sky-limited → **Ninox wins** (SNR 28 vs SCION 23) because its 0.6–1.7 µm band
  excludes the bright visible sky — the SWIR daytime-SDA advantage the paper is built around.

So "detector-only isolation" is strictly true only in the night/signal-limited regime. At daytime levels
the atmosphere (specifically the SWIR sky suppression) is the dominant differentiator, not the detector.
This is a property of the physics, surfaced rather than fixed, per the build instructions.

## 6. Known simplification — sky ratio vs the paper's ~20× benchmark

A pure λ⁻⁴ shape cannot be tuned by normalization to a specific band-integrated ratio; the achieved
SWIR(0.9–1.7)/visible(0.4–0.7) sky ratio is **~11×**, versus the paper's disclosed ~20×. Order-of-magnitude
consistent, but not exact — steepening the exponent or adding the omitted Mie/absorption structure would
be needed to hit 20× and was deliberately not done (the prompt specifies λ⁻⁴). Recorded as a simplification.

## 7. Boundaries honored

- DIRSIG not installed, downloaded, or invoked.
- No numeric value invented for any null/placeholder field — named-placeholder pattern + loud flags used
  throughout.
- The Ninox ADU/e− discrepancy was **not** resolved by guessing a gain.
- `docs/architecture/current.md`, `docs/adr/*`, `docs/audits/0001-*`, and the three prior notebooks were
  not modified. SatSim and toy_model source were not touched.
- Committed locally on `rnd/detector-comparison-benchmark`; nothing pushed.
