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

---

## 8. Extension — SCION-SR daytime-custody demonstration (AMOS §5)

Added 2026-08-06, in place in the same notebook (single source of truth for the model). Supports the AMOS
paper §5 argument that the SCION-SR (VisGaAs) focal plane enables ground-based **daytime/twilight custody**
of LEO objects. Investigation-record tier, same as the rest of this note. Branch: `rnd/scion-daytime-custody`.

### 8.1 Passband capability added to the model
`signal_electron_rate` and `run_detector` gained an optional longpass `lo_cut_um` (applied as a wavelength
mask to **both** the signal and sky-background integrals) plus, on `run_detector`, an `E_tgt_override` for
running an alternate target geometry. Both default to the prior behaviour (`None`), so the existing
detector-comparison section is unchanged — regression-verified: **full-band SCION still SNR 23.2** at the
current daytime `SKY_VIS_REF=20`. The demonstration cutoff is a notebook parameter; **no** config field was
added (the SCION entry deliberately keeps no `effective_bandpass_um`).

### 8.2 Daytime demonstration
`ACTIVE_DETECTOR = "scion_visgaas"`, detection criterion `SNR_DETECT = 5`. Two targets under the fixed
optics/atmosphere: **baseline** 1U @ 500 km (~mag 8.3, the conventional ~10 cm LEO catalog threshold) and a
**harder** 1U @ 1500 km (~mag 10.7, a low-elevation pass), across three sky levels.

- **Result:** the baseline object stays detected across the whole daytime range (SNR ≈ 55 / 23 / 11 at
  `SKY_VIS_REF` = 2 / 20 / 100). The harder 1500 km pass is detected only at the lowest sky (SNR 7.3,
  margin +2.3) and is **lost** by intermediate sky (SNR 2.6, margin −2.4) — the practical daytime custody
  boundary for the fainter geometry. Continuous-sweep SNR=5 crossings: harder ~`SKY_VIS_REF` 5, baseline ~476.
- The budget is sky-dominated at these levels; the fully shot/read-limited regime appears only at
  still-lower sky (shown on the continuous axis of Figure 1 and consistent with the night case in §11 of the
  notebook).

### 8.3 New placeholder values introduced (all flagged loudly in-notebook)
| quantity | value | status |
|---|---|---|
| Three daytime sky levels | `SKY_VIS_REF` = 2 / 20 / 100 W m⁻² sr⁻¹ µm⁻¹ | **placeholder** visible-sky levels — **NOT** a solar-elevation model; no elevation→radiance relation derived |
| Harder target geometry | 1U @ 1500 km slant range (else identical to baseline `SCEN`) | **placeholder** low-elevation pass |
| Detection criterion | `SNR_DETECT = 5` | assumed threshold |
| Passband sweep range / reference | 0.4–1.4 µm swept; ≥1.1 µm cited as reference | reference cutoff from Waldmann, **not** derived here |

The §5–§7 caveats of this note (blackbody solar proxy, placeholder BRDF geometry, λ⁻⁴ sky with Mie=0 /
airglow=0, ~11× vs ~20× SWIR/visible suppression, assumed optics throughput, 1 ms / 5 px) all carry over
unchanged.

### 8.4 Finding — the passband optimum is atmosphere-limited, NOT a publishable result
A cutoff sweep (0.4–1.4 µm, bright sky) shows a broad SNR peak near **~0.74 µm** in this model. **This is not
a sensor-design optimum and must not be reported as one.** Its location is governed entirely by the
placeholder atmosphere — pure λ⁻⁴ Rayleigh with Mie/aerosol = 0 and airglow = 0, achieving only ~11×
SWIR/visible sky suppression versus the literature ~20×. A more complete atmosphere would move the peak.
The sweep is presented as **sensitivity behaviour only**; the operational reference cutoff is Waldmann's
**≥1.1 µm** longpass, and the real optimum is deferred to the atmosphere-complete follow-on. This is the
same class of "atmosphere becomes load-bearing" finding as §5 of this note.

### 8.5 Figures
Four publication-grade figures written to `data/outputs/figures/` (PDF + 300-dpi PNG, AMOS column widths,
manifest alongside): detection-SNR-vs-sky (primary), per-wavelength budget, noise-budget composition, and
the passband sweep (annotated with the atmosphere caveat and the ≥1.1 µm reference). No local AMOS template
was found in the repo, so conference-standard widths (single 3.4 in, double 7.0 in) were used.

### 8.6 Boundaries honored (this extension)
No continuous solar-elevation model and no MODTRAN/Mie/aerosol/airglow (deferred follow-on). Passband
optimum presented as behaviour, not a result. No invented values. Existing comparison outputs unchanged
(regression-gated). `docs/architecture/current.md`, `docs/adr/*`, `docs/audits/*`, and the other three
notebooks untouched; no `src/` extraction. Committed locally on `rnd/scion-daytime-custody`; nothing pushed.
