# Prompt: SCION-SR Daytime-Custody Demonstration (AMOS paper, §5)

## Context

We are writing an AMOS paper whose §5 demonstrates the utility of the SCION-SR (VisGaAs) focal plane for **ground-based daytime/twilight custody** of LEO resident space objects. The radiometric engine already exists in `notebooks/radiometric_detector_comparison.ipynb` (the Waldmann et al. 2025 Appendix-A parametrization, eqs 1–11), validated for the Ninox/SCION/toy comparison. This task extends that notebook with a SCION-SR-only daytime demonstration and adds one missing capability — a selectable observing passband — plus publication-quality figures.

This is a **first-foray** paper. It demonstrates daytime detectability at one-to-three discrete solar-elevation sky levels. It does **not** build a continuous solar-elevation sky model or a real (MODTRAN) atmosphere — those are deferred to a follow-on paper and are hard stops below.

## Scope (one task; do not exceed)

Extend the existing `notebooks/radiometric_detector_comparison.ipynb` in place with a new, clearly delimited section titled **"SCION-SR daytime-custody demonstration."** Reuse the model functions already defined in that notebook; do not re-implement the radiometric chain and do not create a parallel notebook or a `src/` module (single source of truth for the model). Keep the existing detector-comparison section and its outputs unchanged.

## Change 1 — add a selectable passband to the model

- Generalize the signal and sky-background integrals (currently in `run_detector` / `signal_electron_rate`) to accept an **optional longpass cutoff** `lo_cut_um`, applied as a wavelength mask to **both** the signal integral and the sky-background integral (reject λ < cutoff in both).
- Default `lo_cut_um` to the detector's full band (no cutoff) so every existing comparison cell produces identical numbers to before. This must be backward-compatible.
- Read the cutoff as a notebook variable in the new section; do not hardcode it inside `run_detector`. (The SCION config already lacks an `effective_bandpass_um`; Ninox has one at `[1.1, 1.7]` — you may add an analogous field to the SCION entry in `configs/detector_configs.json` if it keeps things config-driven, but the demonstration cutoff itself stays a notebook parameter.)

## Change 2 — the daytime-custody demonstration section

Set `ACTIVE_DETECTOR = "scion_visgaas"`. Define a detection criterion `SNR_DETECT = 5` (single, named, near the top of the section).

- **Baseline target:** the 1U-class LEO object already in the notebook (10 cm characteristic size, ~500 km, the existing placeholder geometry), which corresponds to the conventional ~10 cm LEO trackable/catalog threshold. Also run **one harder geometry** — a longer slant range (e.g. 1500 km, a low-elevation pass) or a 0.5 U target — so the demonstration is not limited to a trivially bright case. Label both.
- **Three discrete daytime sky levels** standing in for increasing solar elevation (twilight / intermediate / high-sun), expressed as `SKY_VIS_REF` values. These are **placeholder visible-sky-radiance levels**, not a solar-elevation model — flag this loudly in a markdown cell. Do not derive a solar-elevation→radiance relation.
- For each (target × sky level): report integrated SNR, the four-term noise budget, and the dominant noise term, in a table. Show the shot-limited (twilight) to sky-limited (bright) regime transition.
- Report detection margin against `SNR_DETECT` for the baseline and harder target across the three sky levels.

## Change 3 — passband sweep (behavioral, NOT an optimum result)

- Sweep `lo_cut_um` across ~0.4–1.4 µm at a bright-sky level and plot SNR vs cutoff.
- **Critical framing (state in a markdown cell):** the SNR-optimal cutoff produced by this model is governed by the placeholder atmosphere (pure λ⁻⁴ Rayleigh, Mie/aerosol = 0, airglow = 0, achieved ~11× vs the literature ~20× SWIR/visible suppression). It is therefore **not a publishable optimum** and must not be reported as one. Present the sweep as sensitivity behavior only, and cite Waldmann's ≥1.1 µm longpass as the operational reference cutoff. The real optimum is deferred to the atmosphere-complete follow-on.

## Change 4 — publication-quality graphic output

Figures are for the AMOS paper and must be publication-grade, not exploratory defaults. Add a style-setup cell and save every figure to `data/outputs/figures/` (create it) in **both** vector PDF and 300-dpi PNG.

- **Sizing:** AMOS/two-column figure widths — single-column ≈ 3.4 in, double-column ≈ 7.0 in. Produce each figure at the width it will occupy in the paper.
- **Style:** consistent `matplotlib` rcParams (serif or the paper's font, axis label + tick font ≥ 8 pt at final size, `figure.dpi`/`savefig.dpi` = 300, `savefig.bbox="tight"`, no title text baked in — captions live in the paper), a colorblind-safe palette, minimal chartjunk, legible legends, axis labels **with units**.
- **Figures to produce:**
  1. Detection SNR vs daytime sky level, for baseline and harder target, with a horizontal `SNR_DETECT` detection line and the three solar-elevation points marked. (Primary figure.)
  2. Per-wavelength budget at a representative daytime point: source-signal and sky spectral electron-rate densities with QE(λ)·T_atm(λ) overlaid (upgrade the notebook's existing plot to publication grade).
  3. Noise-budget composition (shot/sky/dark/read) across the three sky levels — grouped or stacked bars.
  4. Passband sweep (SNR vs cutoff), annotated with the atmosphere-limited caveat and the ≥1.1 µm reference line.
- Save a small manifest (figure filename → one-line description) alongside the figures.

## Documentation

Append a section to `docs/recon/0002-detector-comparison-benchmark.md` (do not start a new note): record the passband addition, the daytime demonstration, the placeholder sky levels, the baseline + harder target, and — explicitly — the finding that the passband optimum is atmosphere-limited and not a publishable result. Same investigation-record tier as the existing note.

## Hard stops — do not cross

- Do **not** build a continuous solar-elevation sky model or a real/MODTRAN atmosphere. Do not add Mie/aerosol or airglow terms. These are the deferred follow-on; they are out of scope here.
- Do **not** report a passband optimum as a result; present the sweep as behavior with the atmosphere caveat.
- Do **not** invent numeric values for anything flagged placeholder; use the named-placeholder pattern and flag loudly (matches the existing notebook's conventions).
- Do **not** change the existing detector-comparison section's outputs; the default (full-band) SCION path must still return SNR 23.2 at the current daytime `SKY_VIS_REF` (regression check).
- Do **not** modify `docs/architecture/current.md`, anything under `docs/adr/`, `docs/audits/`, or the other notebooks (`using_satsim`, `geometry_and_visualization`, `band_integration_validation`).
- Do **not** push to any remote. Work on a new branch (e.g. `rnd/scion-daytime-custody`), commit locally, one commit per logical step.
- Do **not** extract the model into `src/` or refactor the notebook's architecture — single, in-place extension only.

## Verification before considering this done

- Notebook executes top-to-bottom on the `Python (satsim-env)` kernel with zero errors (`nbconvert --execute`).
- Regression: the existing comparison section still yields the prior SNR table (full-band SCION SNR 23.2 at the current daytime sky level).
- All figures written to `data/outputs/figures/` in PDF + 300-dpi PNG at the specified widths.
- Report back: the daytime SNR / detection-margin table across the three sky levels for both targets, the list of placeholder values introduced, and confirmation that the passband optimum is presented as behavior-only per the atmosphere caveat. Short status summary, not a new status document.
