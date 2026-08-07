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

---

## 9. Extension — SCION-SR daytime-custody *bounding model* (AMOS §5)

Added 2026-08-07, in place in the same notebook. Reorients the daytime section (§8) from a detection
*demonstration* into a **detector-parameterized bounding model** for §5. Branch: `rnd/scion-daytime-custody`.
Investigation-record tier. All prior sections unchanged; regression intact (full-band SCION-SR SNR 23.2).

### 9.1 Realistic QE(λ) — the key pending input
The flat-at-peak QE cannot support a band-selection argument, so this section uses **wavelength-resolved
QE placeholders**: `QE_VISGAAS_PARAMETRIC` (SCION-SR: poor in the visible, rising through the NIR, peak
~0.85 across ~1.0–1.5 µm, roll-off to 1.7 µm) and `QE_INGAAS_PARAMETRIC` (reference SWIR sensor: negligible
below ~0.9 µm, peak ~0.85 at ~1.3 µm). No digitized datasheet QE exists in the repo (`qe_curve` null for
all detectors), so these are **hand-built parametric placeholders**, added via a backward-compatible
`qe_override` argument on `run_detector`/`signal_electron_rate` (default `None` = prior flat-at-peak
behaviour). **The measured QE curve is the key pending input** — the band-selection and color results
depend on the QE shape, so the *direction* is robust but the *quantitative optimum* is pending.

### 9.2 Intelligent NIR–SWIR band (method + direction, not a number)
Sweeping the lower cutoff with the realistic SCION-SR QE confirms the **robust direction**: reject the
visible, keep the NIR–SWIR. The model's optimal lower cutoff sits near the visible/NIR boundary (~0.7–0.8
µm) — the payoff is discarding the low-QE, λ⁻⁴-bright **visible**. It is **bluer** than Waldmann's
operational **≥1.1 µm** longpass because this section's placeholder atmosphere (pure λ⁻⁴, no water/thermal
structure — deferred per hard stops) imposes no penalty on 0.8–1.1 µm. We therefore **adopt ≥1.1 µm
(Waldmann) as the intelligent band** for the headline claim and treat the exact optimum as
placeholder-QE/atmosphere dependent — a direction, not a publishable value.

### 9.3 Limiting sensitivity reparameterized by the detector noise floor
**Limiting sensitivity** = faintest target detectable at `SNR_DETECT` for a reference exposure, reported as
limiting apparent magnitude and smallest characteristic size at a reference range (inverting the SNR
relation; the sky/dark/read terms are target-independent). Reference values (new **placeholders**):
`T_REF_S = 10 ms`, `RANGE_REF_KM = 1000 km`. Scalings shown explicitly: SNR ∝ √t in the sky/dark-limited
regime, flux ∝ 1/range² (so limiting size ∝ range).

- **Primary figure — vs read noise** (sky-suppressed, t = 10 ms → read-limited): SCION-SR (30 e⁻) and the
  reference sensor (35 e⁻) are **near-parity** — read noise is not the differentiator here.
- **Companion — vs dark current** (sky-suppressed, t = 1 s → dark-limited): SCION-SR's far lower dark
  current, a consequence of **deep thermoelectric cooling (−70/−75 °C)**, reaches substantially fainter
  targets. SCION-SR's read/dark specs are datasheet **upper bounds**, so the advantage is conservative.
- **Causal chain:** the SCION-SR − reference limiting-magnitude gap is ~0 when sky-limited and **widens as
  the sky is suppressed** — +0.12 mag read-limited (t = 10 ms) vs **+0.62 mag dark-limited (t = 1 s)**. This
  is the paper's sky-limited → detector-limited transition, and it identifies **dark current (deep
  cooling)** as SCION-SR's decisive advantage in the sky-suppressed daytime regime.

### 9.4 NIR–SWIR color index (secondary)
An illustrative NIR–SWIR color (magnitude difference between ~1.0–1.2 µm and ~1.4–1.7 µm sub-bands, both
inside the VISGaAs good-QE region; the visible is excluded because VISGaAs QE is poor there) separates two
toy material slopes. Supports material discrimination, not the §5 custody claim; also placeholder-QE
dependent.

### 9.5 Anonymization and the §10 common-band correction
All competitor references are anonymized **at the display layer only** (config keys left intact): a
`DISPLAY_NAME` map yields **"SCION-SR"** and **"reference SWIR sensor"**, applied to every paper-facing
print, the §10 table, the §11 print, and figure titles; no competitor brand name appears in any rendered
output (verified by scan). The toy model is dropped from the §10 claim-facing table. The §10/§11
interpretation is **corrected**: the earlier daytime SNR ordering was credited to the detector, but it is a
**band-coverage effect** (0.6–1.7 vs 0.4–1.7 µm default bands); a detector-only comparison requires a
**common band**, which this bounding model uses.

### 9.6 Graphics and boundaries
Rich **inline** plots only (light readability rcParams, colorblind-safe palette) — **no** file export,
figures directory, manifest, AMOS sizing, or `.gitignore` change this task (the prior §8/Change-4 export
apparatus is left untouched but not extended). No continuous solar-elevation or MODTRAN/Mie/airglow model
(deferred). New placeholders introduced: the two parametric QE curves, `T_REF_S = 10 ms`,
`RANGE_REF_KM = 1000 km`, the color sub-band edges and toy material reflectance slopes. Committed on
`rnd/scion-daytime-custody`; nothing pushed.

---

## 10. Extension — Full-well/saturation axis and iso-curve utility plots (AMOS §6)

Added 2026-08-07, in place in the same notebook. Promotes the bounding model (§9) to the paper's
**daytime-custody utility study (§6)**: adds **full-well saturation** as a limiting factor and recasts every
claim plot as an **iso-curve family over the detector parameter space** with **no single named competitor**.
Branch: `rnd/scion-daytime-custody`. Investigation-record tier. All prior sections unchanged; regression
intact (full-band SCION-SR SNR 23.2, reference sensor 28.2 — cells 0–50 verified byte-identical in both
source *and* rendered output).

### 10.1 The saturation mechanism
Per-pixel accumulation over the integration is now modelled explicitly: `well_fill_rate` = **sky + dark +
source signal spread over `N_PIX_PSF`**, with `accumulated_e`, `is_saturated`, `t_saturate`, and
`limiting_mag_capped` layered *additively* on the §9.3 solver (`limiting_mag`/`limiting_size_m` reused
unchanged; `run_detector`'s vestigial signal-only `saturated` flag left untouched because the benchmark
section is regression-gated). The largest non-saturating exposure is
**t_sat = f_usable · W_full / fill_rate**, and limiting sensitivity is evaluated at
**t = min(t_requested, t_sat)** — never at an arbitrary fixed time.

Two **idealizations, flagged in-notebook, not given invented numbers**: `WELL_USABLE_FRACTION = 1.0` treats
the entire well as usable (no anti-blooming headroom, no linearity derating; a real ROIC would use ~70–90 %),
and the signal term assumes the PSF spreads charge **evenly** over `N_PIX_PSF`, so a real peaked PSF fills
its central pixel faster and the caps below are optimistic on the signal term.

### 10.2 Result — the well → integration-cap → limiting-sensitivity chain
At the operational band ≥1.1 µm on the reference daytime sky (`REF_SKY_DAYTIME = 20`), the per-pixel
background fill rate is **1.628 × 10⁶ e⁻ px⁻¹ s⁻¹** (sky 1.628 × 10⁶; dark 15 — five orders of magnitude
smaller). The bright baseline 1U target itself would add 4.74 × 10⁵ e⁻ px⁻¹ s⁻¹; a *limiting* target adds
essentially nothing, so the background rate is the one that sets the cap.

| full well (e⁻/px) | t_sat, background (s) | t_sat w/ 1U target (s) | limiting mag @ t_sat | limiting size @ 1000 km (m) |
|---|---|---|---|---|
| 1 × 10⁴ | 0.00614 | 0.00476 | 11.06 | 0.057 |
| 1 × 10⁵ | 0.0614 | 0.0476 | 12.36 | 0.031 |
| **1 × 10⁶ (SCION-SR)** | **0.614** | **0.476** | **13.62** | **0.018** |
| 1 × 10⁷ | 6.14 | 4.76 | 14.87 | 0.010 |

The cap is reached **in the sky-shot-dominated regime**, verified numerically at t_sat and printed in the
notebook: sky shot **99.686 %** of the variance, source shot 0.223 %, read 0.090 %, dark 0.001 % — so
**read + dark together are 0.09 %, i.e. negligible.** That is what licenses the analytic scaling: with
N₀² ∝ t and S_base ∝ t, limiting magnitude = const + **1.25·log₁₀ t**, and t_sat ∝ W_full, hence

> **≈ +1.25 mag of daytime limiting sensitivity per decade of well depth** (measured +1.27 mag/decade across
> the swept family; limiting size shrinks ×0.56 per decade). Over the 3-decade family the achievable
> limiting magnitude on the reference daytime sky moves **11.06 → 14.87**.

This is the mechanism by which a large well converts into deeper daytime reach, and it makes **full well —
not the noise floor — the binding constraint in bright daytime sky.** SCION-SR's 1 M e⁻ well yields a
**0.614 s** non-saturating exposure and limiting mag **13.62** at the reference sky. Absolute magnitudes
inherit the placeholder sky levels; **the scaling is the result, not the absolute numbers.**

### 10.3 Shift to iso-curve families (competitor space, not a single reference point)
The §9 two-point figures are **gone**. `REFSENSOR`, `QE_INGAAS_PARAMETRIC`/`QE_INGAAS_ANCHORS`, the InGaAs
curve on the QE plot, the two-detector print, and the SCION-SR − reference **gap** metric (intrinsically
two-system, unsalvageable) were all removed from the section. Each claim plot is now a family spanning a
representative range of SWIR-detector metrics, with **only SCION-SR's operating point marked** — the family
*is* the competitor space, so no one system is plotted, labelled, or tabulated. Verified by scan: **zero**
occurrences of `REFSENSOR` / `QE_INGAAS` / "InGaAs" / "reference SWIR" in the section's sources or outputs,
and zero brand names in any rendered output notebook-wide (config keys in cells 4/6/24/27 remain, per the
display-layer-only anonymization policy). The frozen §§10–11 cells keep their anonymized reference-sensor
comparison because they are regression-gated.

- **Fig A (headline)** — limiting sensitivity vs *requested* integration time, one iso-curve per full well
  (10⁴–10⁷ e⁻), at the reference daytime sky. Each well follows the common √t improvement until **its own
  saturation knee**, then plateaus: asking for more exposure buys nothing once the well is full. Knees
  marked; SCION-SR's 1 M e⁻ curve highlighted and its knee annotated. **This is the daytime-custody figure.**
- **Fig B (supporting, low-sky/faint only)** — limiting sensitivity vs dark current, one iso-curve per read
  noise (5–100 e⁻), sky **fully suppressed**, t = 1 s. Labelled in its title, lead-in, and printed output as
  the **low-sky / faint-target floor — explicitly NOT a bright-daytime claim**; the printed regime check
  confirms the well is nowhere near limiting there (smallest cap 200 s vs t = 1 s).
- **Fig C (supporting)** — limiting sensitivity vs sky level (200 → 0), one iso-curve per dark current
  (1–5000 e⁻ px⁻¹ s⁻¹), each point evaluated at `min(1 s requested, t_sat)` so the saturation cap is carried
  through. The curves **collapse** at bright sky (spread **0.00 mag** at sky = 200 — sky-limited, detector
  irrelevant) and **fan out** as the sky is suppressed (**1.00 mag** at sky = 0 — detector-limited). Same
  sky-limited → detector-limited causal chain as §9.3, now in absolute sensitivity for one system.
- **Fig D (method/direction only)** — band-selection sweep in limiting sensitivity vs lower cutoff, one
  iso-curve per sky level, with QE(λ) and normalized sky spectral density overlaid. The
  **atmosphere-limited caveat of §9.2 is retained verbatim in substance**: the model optimum (0.70–0.78 µm
  across the sky family) is bluer than the operational ≥1.1 µm because the placeholder atmosphere has no
  water/thermal structure, so **no passband optimum is reported as a result** — ≥1.1 µm (Waldmann) remains
  the adopted band. The fixed 10 ms exposure is confirmed non-saturating everywhere in that sweep (smallest
  cap 0.0441 s).

### 10.4 Swept ranges and SCION-SR's operating point
A printed table replaces the former two-detector comparison: read noise **5–100** (SCION-SR **30**, upper
bound), dark current **1–5000** (SCION-SR **15** e⁻ px⁻¹ s⁻¹, upper bound, deep TEC), full well
**10⁴–10⁷** (SCION-SR **10⁶**, 100 % usable per the idealization), QE(λ) not swept (VISGaAs parametric
placeholder, peak 0.85), pixel pitch fixed at 10 µm, sky **0–200** (reference 20, placeholder levels with no
solar-elevation mapping), requested integration **1 ms–10 s** (cap 0.614 s at the reference sky), band lower
cutoff **0.40–1.41 µm** (adopted 1.10). **No competitor row.** New placeholders this task:
`T_MAX_REQUEST_S = 1 s`, `T_LONG_S = 1 s`, the four swept families, and `WELL_USABLE_FRACTION = 1.0`.

### 10.5 Graphics, one correction, and boundaries
Rich **inline** plots only — no file export, figures directory, manifest, AMOS sizing, or `.gitignore` change
(the §8 export apparatus remains untouched and unextended). Ordered families use a perceptual (viridis) ramp
so the parameter ordering reads, with SCION-SR marked in an accent colour; the legend frame is re-enabled in
the section's rcParams because the publication block's `legend.frameon: False` let iso-curves show through
legend text.

**One correction carried over from §9:** those figures called `ax.invert_yaxis()` while labelling the axis
"fainter ↑", which put fainter magnitudes at the *bottom* — the axis direction contradicted its own label.
The rewritten figures drop the inversion and label the axis "fainter, better ↑", so fainter is genuinely
upward. No other §9 result is affected (the inversion was cosmetic).

Boundaries honored: no continuous solar-elevation model, no MODTRAN/Mie/aerosol/airglow (all still
deferred); no invented QE — the flagged parametric placeholder stands until the measured curve arrives; no
brand/product names and now **no single competitor point at all**; prior sections unchanged and
regression-gated; no `src/` extraction or refactor. Committed on `rnd/scion-daytime-custody`; nothing pushed.
