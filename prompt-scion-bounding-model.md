# Prompt: SCION-SR Daytime-Custody Bounding Model (AMOS §5)

## Context

Continuing on branch `rnd/scion-daytime-custody` (Changes 1–4 already committed: the passband parameter `lo_cut_um`, the daytime-demonstration section, and inline figures). This task reorients the daytime section from a detection *demonstration* into a **detector-parameterized bounding model** for the paper's §5. The paper lays claim to ground-based daytime SWIR custody with the SCION-SR VISGaAs focal plane; the model's job is to **bound limiting sensitivity as a function of the detector noise floor**, and to select an **intelligent NIR–SWIR observing band**. It is a bounding/positioning model, not an exhaustive analysis.

Two physical premises drive the reorientation:

1. **The visible band is doubly disadvantaged for a VISGaAs detector.** The daytime Rayleigh sky is brightest there (λ⁻⁴), *and* VISGaAs QE is poor in the visible while good in the NIR–SWIR. The intelligent band therefore rejects the visible and keeps the NIR–SWIR, where detector QE, target (solar-panel) reflectance, and sky suppression all favor detection. Waldmann et al. 2025 discuss band-limiting (≥1.1 µm longpass) as the operational reference.
2. **Once the band suppresses the sky, the residual limit is the detector noise floor.** That is the regime in which read noise and dark current — the SCION-SR advantages — set limiting sensitivity, so the model must be parameterized by those metrics.

## Scope (one task; do not exceed)

Extend `notebooks/radiometric_detector_comparison.ipynb` in place with a clearly delimited section **"Bounding model for daytime custody (AMOS §5)."** Reuse the existing model functions and the `lo_cut_um` parameter. Keep every prior section byte-for-byte in behavior; existing edits stay backward-compatible and are regression-gated (full-band SCION-SR = SNR 23.2 at the current daytime sky level). New commits on the same branch, one per change below, no push.

## Change A — realistic QE(λ), not flat-at-peak

The flat-at-peak QE assumption cannot support a band-selection argument and must be replaced for this section.

- Add a wavelength-resolved QE(λ) for SCION-SR (VISGaAs) that reproduces the datasheet's qualitative shape: **low in the visible (~0.4–0.7 µm), rising through the NIR, peaking ~0.85 in the ~1.0–1.5 µm NIR–SWIR, rolling off toward 1.7 µm.** If a digitized datasheet QE curve exists in the repo, use it; otherwise implement a **named parametric placeholder** (e.g. `QE_VISGAAS_PARAMETRIC`) that matches this shape, flag it loudly as a placeholder pending the measured curve, and store its defining points/parameters in a clearly-labeled cell (or as a `qe_curve` entry in `configs/detector_configs.json`, keeping the file the single source).
- Give the unnamed reference SWIR sensor an analogous InGaAs-shape QE(λ) (negligible below ~0.9 µm, peak ~1.3 µm, to 1.7 µm), same placeholder discipline.
- State in markdown that the band-selection and color results depend on this QE shape, so the direction (reject visible, keep NIR–SWIR) is robust but the quantitative optimum is pending the measured curve.

## Change B — intelligent NIR–SWIR band selection

- Using the realistic QE, sweep the observing band (lower cutoff, and optionally an upper edge) and report the band that maximizes limiting sensitivity at a reference daytime sky and exposure.
- Plot the sweep with QE(λ) and the sky spectral density overlaid so the trade is visible (sky rejected below the cutoff vs signal retained above it).
- Note explicitly that the realistic QE moves the optimum into the NIR–SWIR (superseding the flat-QE result, which placed it in the visible as an artifact), consistent with Waldmann's ≥1.1 µm longpass. Cite Waldmann as the operational reference; caveat that the exact optimum depends on the placeholder QE and sky model.

## Change C — limiting sensitivity as the headline, parameterized by the detector noise floor

- Define **limiting sensitivity** as the faintest detectable target at `SNR_DETECT` (report as limiting apparent magnitude and/or smallest detectable cross-section at a reference slant range), for a stated reference exposure. Invert the SNR relation (root-find on target brightness/size).
- **Primary claim figure:** limiting sensitivity vs detector **read noise** (and a companion vs **dark current**), evaluated at the intelligent NIR–SWIR band and reference sky/exposure. Mark SCION-SR's operating point and the unnamed reference sensor's datasheet point.
- Show the causal chain the paper argues: as the band suppresses the sky (or as exposure shortens), the limit moves from sky-limited to detector-limited, and the SCION-SR advantage widens. A small panel showing the SCION-vs-reference gap growing as sky is suppressed makes this explicit.
- Treat exposure as a parameter with a stated reference value; note the √t scaling in the sky-limited regime rather than fixing 1 ms silently.

## Change D — anonymize the comparison and fix the block-10 confound

- In every paper-facing output (tables, figures, captions), label detectors generically: **"SCION-SR"** and **"reference SWIR sensor"**. Do not print any brand or product name (no "Ninox," "Raptor," etc.). Drop `toy_model` from all claim-facing outputs.
- Any detector-to-detector evaluation must use a **common observing band** so it isolates the detector. Correct the existing §10 markdown, which currently attributes a band-width effect (one detector admitting less sky) to the detector itself — that is a band difference, not a detector property, and must be relabeled accordingly.

## Change E — NIR–SWIR color index (brief, secondary)

- Illustrate a **NIR–SWIR color observable** (ratio or magnitude difference between a NIR sub-band and a SWIR sub-band, both inside the VISGaAs good-QE region). Motivate it from material discrimination and note the visible is excluded because VISGaAs QE is poor there. Keep this short and illustrative; it supports the discrimination application, not the §5 custody claim.

## Change F — graphics: rich and clear, NOT publication-formatted

- Produce rich, clearly readable **inline** plots: labeled axes with units, legends, a colorblind-safe palette, legible on screen. A light `rcParams` cell for readability is fine.
- **Do not** export figures to PDF/PNG, do not set AMOS/column sizes, do not write a figures directory or manifest, and do not change `.gitignore`. Figures are cut from the notebook at paper-composition time. (This removes the Change-4 export machinery from the prior task; leave any already-written figure files in place but add no new export apparatus.)

## Documentation

Append to `docs/recon/0002-detector-comparison-benchmark.md` (same investigation tier): the realistic-QE placeholder and its dependence, the intelligent NIR–SWIR band result, the limiting-sensitivity reparameterization and the read-noise/dark-current sweep, the anonymization, and the §10 common-band correction. Flag the measured QE curve as the key pending input.

## Hard stops — do not cross

- No continuous solar-elevation sky model; no MODTRAN/Mie/aerosol/airglow. These remain deferred.
- No brand/product names for any competitor in any output — it is "the reference SWIR sensor."
- Do not invent a measured QE curve; use the flagged parametric placeholder (or a digitized datasheet curve if present) and say which.
- Existing sections unchanged and regression-gated (full-band SCION-SR = SNR 23.2 at the current daytime sky level).
- Do not modify `docs/architecture/current.md`, `docs/adr/*`, `docs/audits/*`, or the other three notebooks. No `src/` extraction or refactor. New commits on `rnd/scion-daytime-custody`, local only, no push.

## Verification before considering this done

- Notebook executes top-to-bottom on `Python (satsim-env)` with zero errors (`nbconvert --execute`).
- Regression: prior sections unchanged; full-band SCION-SR still SNR 23.2 at the current daytime sky level.
- The limiting-sensitivity-vs-noise-floor claim figure is produced, with SCION-SR and the unnamed reference sensor marked, at the intelligent NIR–SWIR band.
- No brand names appear in any output cell.
- Report back: the intelligent band (with the QE-placeholder caveat), the limiting-sensitivity figure description, SCION-SR's position vs the reference sensor, and the list of placeholders introduced. Short status summary, not a new status document.

## Decisions already made (do not re-litigate)

- Headline metric is limiting sensitivity, not SNR at a fixed target.
- Competitor is anonymized.
- Band is NIR–SWIR, selected by the model (not a fixed cutoff), with Waldmann's ≥1.1 µm as the reference.
- Graphics are rich inline plots only; no formatted export.
