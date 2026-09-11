# Context transfer — `radiometric_detector_comparison.ipynb`

**Purpose of this file:** a self-contained brief for pasting into a fresh claude.ai session. It describes
one notebook — `notebooks/radiometric_detector_comparison.ipynb` — through a **systems-analysis lens**:
what system it models, which variables it lets you manipulate, what it reports, and what its results do
and do not support. It assumes no access to the repo.

---

## 1. The use case being modeled

A **detector trade study for an electro-optical space-domain-awareness (SDA) sensing system**. The
question the notebook exists to answer:

> *Holding the target, optics, and atmosphere fixed, which focal-plane detector gives the best integrated
> signal-to-noise ratio (SNR) on a resident space object (RSO), and which noise term limits each one?*

It isolates the **detector's** contribution to SNR by feeding three candidate detectors through an
**identical** scene. It is a parametric implementation of the Waldmann et al. 2025 (AMOS) Appendix-A
radiometric model (eqs 1–11) — **not** a reproduction of that paper (see §6).

## 2. The system it represents

A wavelength-resolved forward radiometric chain, evaluated on a 0.4–1.7 µm grid at 1 nm resolution:

```
solar spectrum → target BRDF (reflectance) → range² falloff → atmosphere (transmission + sky radiance)
   → telescope aperture & optics throughput → detector QE(λ) → photon/electron rate
   → noise budget (shot + dark + sky + read) → integrated SNR
```

Each stage is a separable block, which is what makes the notebook a systems-analysis instrument: you can
change one block (e.g. the detector, or the sky level) and watch the SNR and the noise budget respond
while everything else is held constant.

## 3. The fixed scenario (what is held constant to isolate the detector)

- **Target:** one representative "conservative" case — a **1U CubeSat in LEO at 500 km range**, diffuse
  reflection dominant (specular glint off). Works out to ~apparent magnitude 8.
- **Optics (from the paper's Table 1):** 0.7 m aperture, f/6.5, 4540 mm focal length, 3-surface
  protected-aluminium mirror train (throughput ≈ 0.94), unobstructed aperture.
- **Atmosphere:** parametric MODTRAN substitute — flat transmission windows at 1.0/1.2/1.6 µm plus the
  visible, and a λ⁻⁴ Rayleigh sky radiance. Selectable **night** (dark sky) or **daytime** sky level.
- **Integration:** 1 ms exposure, point-source spread over 5 pixels.

## 4. The three detectors (the swap)

Read from `configs/detector_configs.json` (never hardcoded). Noise is used in the **electron domain**
(eq 10 is defined in photo-electrons):

| detector | band (µm) | QE | read noise | dark current | full well | notes |
|---|---|---|---|---|---|---|
| **Ninox 1280** (InGaAs) | 0.6–1.7 | 0.90 peak | 35 e⁻ | 2000 e⁻/s (−15 °C) | 10 000 e⁻ (hi-gain) | SWIR-only band |
| **Teledyne SCION** (VisGaAs) | 0.4–1.7 | 0.85 peak | <30 e⁻ | <15 e⁻/px/s | 1 000 000 e⁻ | noise values are **upper bounds** |
| **toy_model** | (none in config) | **placeholder 1.0** | 15 e⁻ | 2 e⁻/s | 4 095 e⁻ | **QE is non-physical** — not comparable |

`qe_curve` is `null` for all three, so QE(λ) is modeled as **flat at peak within the detector's band**.

## 5. Levers and gauges (the systems-analysis interface)

**Levers (inputs to vary):**
- `ACTIVE_DETECTOR` — single switch to swap detector (`ninox_1280` / `scion_visgaas` / `toy_model`).
- `SKY_VIS_REF` — daytime sky brightness (drives the night-vs-day regime).
- Target geometry (`SCEN` dict: area, albedo, range, phase, Earthshine, specular alignment/glint).
- Integration time, PSF footprint, optics parameters.

**Gauges (outputs):**
- **Integrated SNR** per detector.
- **Per-term noise budget** — shot / dark / sky / read, in electrons, so you can see *which subsystem
  limits* each detector.
- **Per-wavelength budget plot** — spectral signal and sky electron-rate densities with QE(λ)·Tₐₜₘ(λ)
  overlaid (where in the band the signal and background come from).
- **Comparison table** across all three detectors under the identical scenario.
- Saturation flag (signal vs. full well), source electron rate, apparent magnitude.

## 6. Key results for systems analysis

Representative numbers (with the placeholders of §3; 1 ms, 5 px):

| regime | Ninox | SCION | toy_model | limiting term |
|---|---|---|---|---|
| **Night** (dark sky) | SNR 61 | **SNR 80** | 101* | shot / read |
| **Day** (bright sky) | **SNR 28** | SNR 23 | 25* | sky |

*toy_model uses a placeholder QE — not physically comparable.

Findings that a systems analysis must carry forward:

1. **There is no single "best detector" — the ranking flips with the sky regime.**
   - *Night / signal-limited:* SCION wins on its **wider band** (more collected signal) plus lower read
     noise.
   - *Day / sky-limited:* **Ninox wins** because its narrower 0.6–1.7 µm band **excludes the bright
     visible sky** — the SWIR daytime-SDA advantage the whole benchmark is about.
2. **The atmosphere is load-bearing at daytime levels.** The premise that "atmosphere is a neutral
   constant, only the detector matters" holds **only when sky is sub-dominant** (night). In daytime the
   sky background is the dominant differentiator, not the detector. (This is a deviation from the task's
   original framing, surfaced deliberately.)
3. **The load-bearing output is *relative*, not absolute.** Absolute SNR values are scenario-dependent to
   within the placeholders (§7). What the notebook establishes reliably is the *ordering and the limiting
   noise term* under a consistent scenario.
4. **Band coverage vs. peak QE is a real trade.** SCION's lower peak QE (0.85) but wider band beats
   Ninox's higher peak QE (0.90) on total signal at night — band width can outweigh peak QE.
5. **Read noise and full-well matter at short exposures.** At 1 ms, read noise is comparable to shot noise
   (e.g. Ninox read 78 e⁻ vs shot 84 e⁻ over the PSF), and Ninox's small 10 k e⁻ well is a dynamic-range
   constraint SCION's 1 M e⁻ well does not have.

## 7. Modeling assumptions and limits (what NOT to trust in absolute terms)

Every value below is a flagged stand-in, because the paper publishes only class names, Table 1 optics, and
one sky-ratio benchmark:

- Solar spectrum = 5778 K blackbody (proxy for the paper's WHI curve).
- Target geometry (area 0.01/0.06 m², albedo 0.20/0.25, 500 km, phase 0.5, Earthshine 0.30) — placeholders,
  **not reverse-engineered** from the paper's SNR results.
- BRDF functional forms — a parametrization of eqs 3–7, not the paper's verbatim equations.
- Atmosphere — λ⁻⁴ sky with **Mie/aerosol = 0 and airglow = 0** (flagged omissions); achieved
  SWIR/visible sky ratio ≈ **11×** vs. the paper's disclosed ~20× (a pure λ⁻⁴ shape can't be tuned to hit
  20× exactly).
- Daytime sky absolute level, mirror throughput (0.94), unobstructed aperture, 1 ms / 5 px — all
  placeholders held fixed across detectors.
- QE(λ) flat-at-peak within band (qe_curve null for all); toy_model QE = **non-physical placeholder 1.0**.

**Consequence:** use the notebook to compare detectors and to reason about *which subsystem limits
performance in which regime* — not to quote an absolute SNR for a real mission without replacing the
placeholders with mission-specific values.

## 8. Open data inconsistency (unresolved by design)

The Ninox config carries two irreconcilable noise sets: paper (ADU) **224.1 ADU read / 205.7 cts/s dark**
vs. datasheet (electrons, −15 °C) **35 e⁻ read / 2000 e⁻/s dark**. No single gain fits both — a gain
matching the read numbers implies a dark current ~50× below the datasheet. The notebook uses the
**datasheet electron values** and does **not** invent a gain to force a fit. Resolving it needs the paper's
actual gain/temperature operating point. (Full detail: `docs/recon/0002-detector-comparison-benchmark.md`.)

## 9. How to run

Conda env **`satsim-env`** (Python 3.10), Jupyter kernel "Python (satsim-env)". Dependencies are numpy,
pandas, matplotlib, astropy — no TensorFlow, so it runs in seconds. Set `ACTIVE_DETECTOR` near the top and
run top-to-bottom; the notebook executes cleanly for all three detector values. Detector parameters live in
`configs/detector_configs.json`.

---

**One-line framing for the new session:** *This notebook is a parametric SNR trade-study of an EO/SDA
sensor front-end that swaps three real SWIR/VisGaAs detectors through one fixed 1U-CubeSat-LEO scene; its
reliable output is the relative detector ranking and the limiting noise term per regime — with the central
finding that the ranking flips between night (band-width-limited) and day (SWIR-sky-suppression-limited),
and with all target/atmosphere/solar inputs as clearly-flagged placeholders.*
