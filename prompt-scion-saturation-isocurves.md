# Prompt: Full-Well/Saturation Axis + Iso-Curve Utility Plots (AMOS §6)

## Context

Continue on branch `rnd/scion-daytime-custody`. The bounding-model section added last task is now the paper's **daytime-custody utility study** (§6). Two changes: add full-well/saturation as a limiting factor, and recast every claim plot as an **iso-curve family over the detector parameter space** with **no single named competitor**. The section's job is to support utility conclusions; keep model description in the notebook minimal — the plots carry the conclusions. Regression-gated (full-band SCION-SR = SNR 23.2 at the current daytime sky), local commits, no push.

## Change 1 — full-well / saturation axis

- Add per-pixel accumulation over the integration time (sky + signal + dark) and flag saturation when accumulated electrons exceed the full well.
- In bright daytime sky the well caps the usable integration time: the maximum non-saturating exposure is set by the sky-fill rate and the well depth. Limiting sensitivity must be evaluated at the largest non-saturating exposure (or the requested exposure if shorter), not an arbitrary fixed time.
- Report, over a range of well depths, the integration time at which the well saturates on the reference daytime sky, and the limiting sensitivity reachable up to that cap. This is the mechanism by which a large well converts into deeper daytime reach.

## Change 2 — iso-curve plots, no single-system comparison

Replace the SCION-vs-reference two-point figures from the last task with iso-curve families spanning a representative range of SWIR-detector metrics. Mark **only** SCION-SR's operating point on each family. Do not plot, label, or tabulate any specific competitor — the family spans the competitor space, so no one system is singled out.

Figures (rich inline plots; no export, no sizing, no manifest):

- **Fig A — limiting sensitivity vs integration time, one iso-curve per full-well / saturation level.** Show the saturation knee where each well caps integration; mark SCION-SR's well (1 M e⁻). This is the daytime-custody figure: larger well → longer non-saturating integration → deeper limiting sensitivity.
- **Fig B — limiting sensitivity vs dark current, one iso-curve per read-noise value,** evaluated at a long (dark-limited) integration. Mark SCION-SR's operating point. Shows the floor at long integration.
- **Fig C — limiting sensitivity vs sky level (or band lower-cutoff), showing the sky-limited → detector-limited transition.** Mark SCION-SR.
- **Fig D — band-selection sweep** (limiting sensitivity vs lower cutoff) with QE(λ) and sky spectral density overlaid; keep the atmosphere-limited/method-only annotation from the last task.
- **Table — parameter ranges swept and SCION-SR's operating point** (read noise, dark current, full well, QE). Anonymized; no competitor row.

Headline band remains the NIR–SWIR (≥1.1 µm operational reference); reference range and exposure as before, with √t and 1/range² scalings noted.

## Hard stops

No continuous solar-elevation model, no MODTRAN/Mie/aerosol/airglow. No passband optimum reported as a result (Fig D is method/direction). No invented QE — the flagged parametric placeholder stands until the measured curve arrives. No brand/product names, and now no single competitor point at all — only the parameter family plus SCION-SR. Prior sections unchanged and regression-gated. Inline plots only. New commits on the branch, no push. No `src/` extraction.

## Documentation

Append to `docs/recon/0002-detector-comparison-benchmark.md`: the saturation/full-well mechanism and the well-vs-integration-cap result, and the shift to iso-curve families (competitor space rather than a single reference point).

## Verification

- `nbconvert --execute` top-to-bottom, 0 errors, `Python (satsim-env)`.
- Regression: prior sections unchanged; full-band SCION-SR SNR 23.2 at the current daytime sky.
- Figs A–D are iso-curve families with SCION-SR marked and no competitor labeled anywhere.
- The well→integration-cap→limiting-sensitivity result is reported (the saturation exposure per well depth on the daytime sky).
- Report back: the saturation-cap result, the three iso-curve figures' descriptions, and SCION-SR's location in the parameter space. Short status summary.
