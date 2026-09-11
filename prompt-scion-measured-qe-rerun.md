# Prompt: Re-run §6 with Measured VISGaAs QE and Dark Current

## Context

Teledyne has delivered measured characterization for the SCION-SR VISGaAs detector at −70 °C — a QE(λ) curve and a dark-current value. Propagate these into the §6 daytime-custody utility study in `notebooks/radiometric_detector_comparison.ipynb`, branch `rnd/scion-daytime-custody`. They replace the parametric-QE placeholder and the datasheet dark-current bound used so far; §6's figures and absolute numbers are regenerated from them.

The measured data is embedded below — no external spreadsheet is needed.

## Scope (one task; do not exceed)

Edit `configs/detector_configs.json` and the §6 section of the notebook so §6 uses the measured QE and dark current, then re-run and report. **Do not modify the detector-comparison section (cells 0–50), `qe_of`, `qe_peak`, or the existing `qe_curve` field** — those feed the regression gate and must stay byte-identical. New commits on the branch, local, no push.

## Change 1 — config (`scion_visgaas`)

- Set `dark_current.value` to `11.2`; update its note to "measured, −70 °C, Teledyne 2026".
- Add a **new** field `qe_curve_measured` (leave `qe_curve` = null and `qe_peak` = 0.85 unchanged, so `qe_of` and the comparison section are untouched):

```json
"qe_curve_measured": {
  "temperature_C": -70,
  "source": "Teledyne measured, VISGaAs 1.7 um cutoff, 2026",
  "wavelength_nm": [300,350,400,450,500,550,600,650,700,750,800,850,900,950,1000,1050,1100,1150,1200,1250,1300,1350,1400,1450,1500,1550,1600,1650,1700,1750,1800],
  "qe": [0.26,0.30,0.3725,0.52,0.65,0.7425,0.795,0.8225,0.835,0.845,0.845,0.8525,0.86,0.8575,0.855,0.855,0.86,0.8625,0.865,0.87,0.865,0.86,0.85,0.84,0.82,0.775,0.40,0.03,0.005,0.0,0.0]
}
```

## Change 2 — §6 QE

In the §6 section, replace `QE_VISGAAS_PARAMETRIC` with the measured curve: load `qe_curve_measured` from the config and interpolate (linear or Akima) onto `GRID_UM`, zero outside its wavelength range. Use it as the SCION-SR QE for every §6 model call — band selection, limiting sensitivity, saturation. Update the §6 markdown so the QE is stated as measured (remove the "parametric QE placeholder / pending measured curve" caveat), but **keep the remaining placeholders**: blackbody solar proxy, λ⁻⁴ sky with aerosol and airglow omitted, assumed throughput.

## Change 3 — re-run and report

Re-run top-to-bottom, then report:

- **The band optimum from the measured QE — but note the atmosphere is still the λ⁻⁴ placeholder (no water/aerosol), so the optimum remains method/direction, not a publishable result.** Keep ≥1.1 µm as the adopted operational reference and label Fig D accordingly. Do not present the re-run optimum as final.
- Updated limiting magnitudes and sizes: the Fig A saturation-knee values per full well, the smallest detectable size at 1000 km, and the 1U-at-1000 km margin.
- Confirm the +1.25 mag/decade-of-well scaling still holds (it is QE-independent).
- Confirm the comparison section still shows SNR 23.2 (dark current is sky-masked there, so 11.2 vs 15 should not move it at displayed precision; if it does, report the new value).

Regenerate Figs A–D with the measured QE (inline plots only; no export, no figures dir, no sizing).

## Hard stops

- Do not touch cells 0–50, `qe_of`, `qe_peak`, or `qe_curve` (null). Regression: comparison section byte-identical, SCION full-band SNR 23.2.
- Do not add water/aerosol/airglow or a real atmosphere — separate future task; the band optimum stays direction-only.
- No brand names in any output; inline plots only; branch commits, no push; do not modify `docs/architecture/current.md`, `docs/adr/*`, `docs/audits/*`, or the other notebooks; no `src/` extraction.

## Documentation

Append to `docs/recon/0002-detector-comparison-benchmark.md`: the measured QE(λ) and dark-current values, the `qe_curve_measured` field, the §6 QE swap, and the refreshed §6 numbers — noting the band optimum is still atmosphere-limited.

## Verification

- `nbconvert --execute` top-to-bottom, 0 errors, `Python (satsim-env)`.
- Regression: comparison section unchanged; SCION full-band SNR 23.2.
- §6 uses the interpolated measured QE and dark current 11.2; Figs A–D regenerated.
- Report: new band optimum (flagged direction-only), new limiting magnitudes/sizes, the 1U margin, and confirmation the well-depth scaling is unchanged.

## Workflow note

Author this as `prompt.md` at repo root and run it planning-first per `CLAUDE.md`: "enter planning mode and produce a plan from `prompt.md`, then stop"; review; then "implement the approved plan, then stop and report back."
