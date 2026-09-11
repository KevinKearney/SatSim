# Prompt: Detector-Comparison Radiometric Benchmark Notebook

## Context

RSO_Sim/SatSim's current architecture is documented in `docs/architecture/current.md` (Rev 2). Separately, we've been benchmarking against Waldmann et al. 2025 ("Infra-Red Sensor Technology Demonstrator System for Space Domain Awareness," AMOS 2025) -- specifically Appendix A's radiometric model (eqs 1-11): a wavelength-resolved photon-rate and SNR calculation combining solar spectrum, a four-term BRDF reflectance model, atmospheric transmission, and a detector noise budget.

Goal: build a notebook that implements this model parametrically and swaps between three candidate detectors -- Ninox 1280, Teledyne Judson SCION (VisGaAs), and our toy_model -- while holding the optical/target/atmosphere scenario fixed at the paper's own values. This isolates the detector's contribution to SNR. It is explicitly NOT a full reproduction of the paper -- the atmosphere and target-BRDF inputs are necessarily simplified/parametrized (see below), and that's a deliberate scoping decision, not a shortfall to fix.

## Scope

Build: `notebooks/radiometric_detector_comparison.ipynb`

Detector parameters: load from `configs/detector_configs.json` (already in the repo). Do not hardcode detector values inline in the notebook -- read this file. It is intentionally incomplete for some detectors (nulls where the underlying source document/config doesn't specify a value) -- see "Handling missing/inconsistent values" below; this is expected, not a bug to fix by inventing numbers.

## Model to implement (Waldmann et al. 2025, Appendix A)

- Eq 1: uniform wavelength grid, lambda_ref = 1um, default R = 1000. Span the grid to cover the union of all three detectors' spectral_range_um (widest is SCION's 0.4-1.7um) so the same grid works when swapping the active detector.
- Eq 2: flux at spacecraft = solar irradiance (no attenuation before the target). Use a blackbody proxy at T=5778K normalized to the solar constant (~1361 W/m^2 at 1 AU) as a stand-in for the paper's WHI reference spectrum -- flag this substitution explicitly in a markdown cell, don't silently treat it as the real WHI curve.
- Eqs 3-7: four BRDF components (specular flat, diffuse body, body+Earthshine, specular+Earthshine) plus the Earth reflectance function. The paper does not publish numeric target geometry (area, albedo, radius, distance) for its four reference targets (1U CubeSat LEO, 12U CubeSat MEO, 12U CubeSat GEO, Skynet-class GEO) beyond the class names. Assume ONE representative scenario -- recommend 1U CubeSat in LEO at ~500km range, the paper's most conservative case -- with clearly-labeled placeholder area/albedo/radius values (e.g. 0.1m x 0.1m body, solar panel area ~0.06 m^2, body albedo ~0.2, panel albedo ~0.25). Do not reverse-engineer these from the paper's Section 7 SNR results -- that's circular. Put these assumed geometry values in one clearly-commented config cell near the top of the notebook, separate from the detector config.
- Eqs 8-9: collected power and photon rate, using QE(lambda) from the active detector's config, telescope throughput and aperture from the paper's Table 1 (0.7m aperture, f/6.5, 4540mm focal length, 3-surface protected-aluminum mirror coating -- assume ~0.98 reflectance/surface, i.e. ~0.94 for 3 surfaces, since the paper doesn't publish an exact coefficient; flag as assumed).
- Eqs 10-11: noise budget and integrated SNR, using the active detector's read noise / dark current / full well from the config.

## Atmosphere (parametric substitute for MODTRAN)

The paper does not publish its MODTRAN transmission curve, water-column-density value, or aerosol model -- only descriptive text and one disclosed benchmark: daytime sky radiance in the 0.9-1.7um band is ~20x lower than in the visible (Sections 2.1/3.1). Build a simple parametric atmosphere instead of running MODTRAN:

- Rayleigh term proportional to lambda^-4, normalized so the SWIR/visible sky-radiance ratio at 1.5um matches this ~20x benchmark.
- Flat transmission "windows" at 1.0, 1.2, 1.6um (named in the paper) with no fine absorption-line structure.
- No Mie/aerosol term, no airglow term -- set both to zero and flag clearly as omitted, not silently folded into the Rayleigh term.

This is a simplification, not a MODTRAN-equivalent. Because the comparison is detector-only, the SAME atmosphere is used for every detector run, so its absolute accuracy is not load-bearing for the comparison's validity -- only its consistency across runs matters. State this reasoning in a markdown cell; don't bury it in a code comment.

## Handling missing/inconsistent values in configs/detector_configs.json -- do not paper over these

- **toy_model** has no qe_peak, qe_curve, or spectral_range_um at all. It cannot be run through eqs 8-9 without one. Do not invent a plausible-looking QE silently. Instead: define a single named placeholder (e.g. `TOY_MODEL_QE_PLACEHOLDER = 1.0`, flat, across the full grid) in its own clearly-labeled cell, print a loud markdown warning when toy_model is selected, and label every plot/table entry for toy_model "QE ASSUMED -- NOT PHYSICAL" rather than presenting its SNR_int as comparable to the other two.
- **ninox_1280** carries two disjoint noise-number sets: `read_noise_paper` / `dark_current_paper` (ADU/cts, as reported in the Waldmann paper, gain unknown) and `read_noise_datasheet_e` / `dark_current_datasheet_e_p_s` (electrons, from Raptor's public datasheet at -15C, which may not match the paper's actual operating temperature). These do not reconcile under any single plausible gain -- a gain that fits the read-noise numbers implies a dark current ~50x below the datasheet's typical value. Use the datasheet's electron-domain values (`read_noise_datasheet_e.hg_typical`, `dark_current_datasheet_e_p_s.typical`) for the noise budget, since eq 10 is defined in photo-electrons -- but print a markdown cell stating plainly that the paper's own ADU-domain numbers could not be used or reconciled, and why. Do not derive a gain factor to force a fit.
- **scion_visgaas** values are published as upper bounds ("< 30 e-", "Target < 15 e-/px/s"), not measurements. Use them as given but label them as bounds in any output table (e.g. column header "read_noise (<, e-)"), not as if they were point measurements.

## Output

1. Primary run: one detector selected via a single variable near the top of the notebook (`ACTIVE_DETECTOR = "ninox_1280"` or `"scion_visgaas"` or `"toy_model"`), full walkthrough of eqs 1-11 for that detector under the fixed scenario, ending in SNR_int and a plot of the per-wavelength noise-budget terms.
2. Comparison table: loop over all three detectors under the identical fixed scenario, tabulate SNR_int (and the underlying photon rate / dominant noise term) side by side. Flag toy_model's row as not physically meaningful per above.

## Documentation

Write a short recon-style note at `docs/recon/0002-detector-comparison-benchmark.md` -- same tier/format as `docs/recon/0001-injection-mechanism-reconnaissance.md` (investigation record, no architecture decision). Cover: what the notebook implements, every assumed/placeholder value introduced (atmosphere, target BRDF geometry, mirror throughput, toy_model QE), and the Ninox ADU/electron discrepancy as an open question. Do not edit `docs/architecture/current.md` -- that stays under Kevin's control.

## Hard stops -- do not cross these under any circumstance

- Do not install, download, or invoke DIRSIG.
- Do not invent numeric values for anything flagged null/placeholder above -- use the named placeholder pattern and flag loudly instead.
- Do not attempt to resolve the Ninox ADU-vs-electron discrepancy by guessing a gain factor.
- Do not modify `docs/architecture/current.md`, any file under `docs/adr/`, or `docs/audits/0001-...` -- those are frozen or Kevin-owned.
- Do not modify `notebooks/using_satsim.ipynb`, `notebooks/geometry_and_visualization.ipynb`, or `notebooks/band_integration_validation.ipynb`.
- Do not touch the toy_model repo or SatSim's own source code -- this is a standalone analysis notebook, not a patch to either.
- Do not push to any remote. Commit locally on a new branch (e.g. `rnd/detector-comparison-benchmark`), one commit per logical step.
- If you find additional gaps or inconsistencies while building this beyond what's listed above, catalogue them in the recon note -- do not stop to fix them and do not let them block finishing the notebook.

## Verification before considering this done

- Notebook executes top-to-bottom on the `Python (satsim-env)` kernel with zero cell errors (`nbconvert --execute`).
- Re-run with each of the three `ACTIVE_DETECTOR` values to confirm no detector-specific crash (toy_model's placeholder path especially).
- Report back: the final SNR_int comparison table, a list of every assumption/placeholder introduced, and the Ninox discrepancy -- as a short status summary, not a new status.md-style document (that pattern is retired per `docs/architecture/current.md` Section 8).
