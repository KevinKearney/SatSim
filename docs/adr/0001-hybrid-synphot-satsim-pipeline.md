# ADR-0001: Hybrid synphot/SatSim/DIRSIG Image-Chain Pipeline

**Status:** Proposed
**Date:** 2026-08-02
**Deciders:** Kevin

## BLUF

This is the R&D phase toward an eventual differentiable model for unresolved-RSO simulation. SatSim is the core element and already meets most of the requirements of a differentiable model, with specific exceptions to be addressed (see Guiding Constraint). DIRSIG and other radiometric codes produce spectral background truth, preprocessed offline into a form the differentiable model can consume directly.

## Context

Training and evaluating RSO detection/tracking algorithms requires labeled electro-optical imagery at volumes and SNR conditions where human/manual annotation is unreliable. No single existing tool covers the full chain from radiometric truth to labeled, sensor-realistic frames to scene-geometry visualization:

- **DIRSIG** provides radiative-transfer-grade radiometric truth (SED, BRDF, atmosphere) but has no ML-training-oriented annotation pipeline and is not built for high-volume labeled-frame generation.
- **SatSim** provides GPU-accelerated, annotated frame generation (geometry, tracking modes, PSF, detector noise, truth files) but treats brightness as a single scalar — no wavelength axis, no native spectral filtering, no zodiacal/background radiometric model, no BRDF/attitude-dependent glint.
- **synphot** provides correct SED × bandpass × QE → magnitude integration (including AB magnitudes) during R&D — used to prototype and validate the band-integration physics — but produces no imagery, has no scene/detector model, and is not GPU- or autodiff-native. It is not expected to carry forward past R&D: the validated functionality is reimplemented as a separate differentiable component, and synphot itself is retired once that component exists. synphot's own container (`SourceSpectrum`/`SpectralElement`) is strictly 1D — one flux(λ) curve at a time — and cannot hold a DIRSIG hypercube directly. **specutils' `Spectrum`/`Spectrum1D`** is the correct R&D-phase container instead: it natively supports N-dimensional flux arrays sharing one spectral axis (the exact shape of a DIRSIG (x, y, λ) cube), and `synphot.Observation` accepts a specutils `Spectrum1D` as its source-spectrum argument. Whether `Observation` batch-integrates across a multidimensional `Spectrum1D` in one call, or only accepts it as an alternate single-spectrum format and still requires per-spaxel iteration, is unconfirmed (see Action Items).
- Geometry visualization (SatViz/Cesium, or a Skyfield/poliastro fallback) is needed to sanity-check scene configuration before committing to expensive renders, independent of the radiometric/annotation chain.

A longer-term goal is to package this capability as a differentiable, NVIDIA GPU-compatible model. This goal is **not** a constraint on the current R&D phase — DIRSIG and parts of SatSim's noise/annotation path are inherently non-differentiable, and forcing differentiability now would block correct tool selection. The goal should instead act as a soft steering preference during implementation.

## Decision

Adopt a **hybrid, separation-of-concerns pipeline** rather than extending or replacing any single tool to cover the whole chain:

1. **Band integration layer** — during R&D, this is synphot (SED × bandpass × QE → magnitude) operating on DIRSIG hypercubes held in a specutils `Spectrum`/`Spectrum1D` container, used to establish and validate the correct physics. Once the differentiable model is built, this functionality is reimplemented from scratch as a differentiable, GPU-native operation living inside the model — not a refactor of the synphot codebase, which is retired at that point. It applies spectral transmission (bandpass, QE) at training time. This is the sole fix for SatSim's missing color term, the sole path to AB-magnitude or SWIR-band correctness, and — by living inside the graph rather than freezing a magnitude file — the mechanism that keeps spectral-response (bandpass/QE/DMD-mask) co-design open as a future capability rather than foreclosing it.
2. **SatSim** — image formation and annotation engine. Consumes literal magnitudes (from synphot, or from its own catalogs/Lambertian model when full radiometric correctness isn't required). Owns geometry (site/track/TLE or RPO relative-motion), PSF, detector noise chain, and truth-file generation.
3. **DIRSIG** — radiometric background and target truth. Runs offline and stores noiseless hypercubes; DIRSIG itself is treated as a fixed, non-differentiable forward model, not something the training process can affect. SatSim injects the RSO onto the DIRSIG background via a mechanism (TBD) that will include variation of target properties over the frame sequence — emulating a light curve (tumble, glint, or other time-varying brightness) rather than a static per-frame magnitude. The injected background must be noiseless and pre-convolved with the system PSF before injection, so that SatSim's own shot-noise draw remains the sole noise source in the chain and background/target noise is never double-counted.
4. **SatViz/Cesium** (primary) or **Skyfield + poliastro/matplotlib** (fallback) — scene-geometry visualization, decoupled from the radiometric chain. Bridge to Cesium is SatSim's native CZML export (`save_czml`); the SatViz/Cesium integration specifics are not yet confirmed and must be verified against source before being relied on.

No component is asked to do another component's job. Each tool is used only where it is the correct physics or engineering tool, and the interfaces between them (hypercube handoff, target-injection/light-curve mechanism, PSF pre-blur, geometry export) are owned explicitly rather than assumed.

### Guiding constraint: differentiability / GPU compatibility

One component is a firm exception to the soft-preference framing below: **the band-integration functionality currently prototyped via synphot must, when reimplemented as its eventual differentiable component, be differentiable and GPU-native** — because it is the only point in the chain where spectral-response parameters (bandpass, QE, DMD-mask shape) could ever be optimized against a downstream objective. This is a decided property of that future, separately-authored component — not a directive to refactor the synphot codebase, and not a directive to begin that reimplementation now. Sequencing is governed entirely by the Action Items list below; as of this writing, this reimplementation is step 4, gated behind the SatSim tutorial notebook, the visualization approach, and R&D validation of the band-integration physics, and should not be started ahead of that order on the strength of this section alone.

Everywhere else, the constraint remains a soft steering preference. Where a method choice is otherwise neutral (comparable effort, comparable clarity, no loss of correctness), prefer implementations consistent with an eventual differentiable, GPU-compatible packaging — e.g., vectorized array operations over ad hoc Python loops, interfaces that could plausibly be swapped for an autodiff-framework equivalent later, avoiding gratuitous CPU-only dependencies. GPU compatibility and differentiability are delivered by the choice of autodiff framework (JAX, PyTorch, or TensorFlow), each of which runs on CUDA as its underlying compute layer — CUDA itself is not a separate implementation option, and no code in this pipeline is expected to be written as hand-authored CUDA kernels.

**Which framework — JAX, PyTorch, or TensorFlow — is not yet decided, and should not be assumed from any single conversation.** Relevant, non-decisive considerations: SatSim is already TensorFlow-native, which favors TensorFlow for anything that needs to interoperate directly with SatSim's own graph; this practice's existing convention elsewhere favors JAX for forward-modeling/Fisher-information work and PyTorch for detection/deployment, which may or may not extend to this component. This choice is deferred to a dedicated decision (see Action Items), not settled here.

This preference **does not**:
- Block use of DIRSIG, or any other non-differentiable tool, where that tool is the physically/architecturally correct choice — DIRSIG remains external, precomputed, and non-differentiable regardless of what happens to SatSim.
- Justify refactoring working exploratory code for differentiability's sake during initial notebook-stage exploration.
- Apply retroactively — R&D-stage code is expected to violate this constraint routinely and does not need to be revisited unless the code is being carried forward into the eventual differentiable model.

The differentiable artifact will be a refactor of SatSim itself, not a separate surrogate trained on this pipeline's output. SatSim's core radiometric/PSF/rendering chain is believed to be TensorFlow-native and differentiable by construction, pending the source audit in Action Items; "addressing the exceptions" (BLUF) currently means patching two identified non-differentiable operations in SatSim's own graph — replacing Poisson noise sampling with a differentiable (Gaussian) reparameterization valid at the photon-count regimes in use, and excluding annotation/truth-file logic from the differentiable graph entirely, since it was never part of the forward radiometric path and never needs a gradient through it. This is a deliberate departure from the "simulate-then-train, no gradient crossing the framework boundary" pattern used elsewhere in this practice (JAX for forward modeling, PyTorch for detection/deployment): here, the simulator itself is the thing being made differentiable, which is why the framework choice (above) matters more here than it would for an external surrogate.

## Options Considered

### Option A: DIRSIG end-to-end
| Dimension | Assessment |
|---|---|
| Radiometric fidelity | High — full radiative transfer, BRDF, atmosphere |
| Annotation/ML infrastructure | None — not built for labeled-volume generation |
| GPU/training-scale throughput | Poor fit |

**Rejected** — wrong tool for the actual deliverable (labeled detection/tracking training data at volume).

### Option B: SatSim end-to-end (no synphot, no DIRSIG)
| Dimension | Assessment |
|---|---|
| Annotation/throughput | Excellent — this is SatSim's native purpose |
| Radiometric correctness | Grey/scalar only — no color term, no physical background |
| Effort | Lowest |

**Rejected as the general solution** — acceptable only for narrow panchromatic, point-source-only work where color and background radiometry don't matter. Not sufficient for spectral, SWIR, or structured-background (e.g., Earth-limb, cloud, terrain) scenarios, which require true per-band radiometry that a single scalar magnitude cannot carry.

### Option C: Hybrid pipeline (chosen)
| Dimension | Assessment |
|---|---|
| Radiometric correctness | Correct at each stage, by construction |
| Annotation/throughput | SatSim's native strength, preserved |
| Integration burden | Real — registration, PSF pre-blur, and background-ephemeris glue are owned by us, not provided by any tool |
| Extensibility | Each layer independently replaceable/upgradable |

**Chosen** — the only option where each tool is used for what it's actually built for, and no tool is quietly relied upon for physics it doesn't model.

## Trade-off Analysis

The hybrid architecture trades a small amount of upfront integration work (magnitude handoff format, background pre-blur discipline, ephemeris indexing, CZML verification) for correctness at every stage and independence from any single tool's roadmap. The alternative — extending SatSim to natively support spectral bands, AB magnitudes, or physical backgrounds — would mean patching a tool whose architecture (scalar brightness, TensorFlow-graph noise chain) is not built to carry that information, for uncertain long-term maintainability gain.

The differentiability/GPU-compatibility constraint is deliberately non-binding during R&D, with one exception: the band-integration functionality (prototyped via synphot during R&D, reimplemented as a differentiable component afterward), held to a hard differentiability requirement as stated in the Guiding Constraint section. Treating the constraint as binding everywhere would either force premature reimplementation of DIRSIG/SatSim internals (not realistic) or bias early tool choices away from the correct physics tool toward a GPU-friendly but wrong one. Treating it as advisory outside this one component keeps the door open for the eventual refactor without slowing down the current build.

Building the band-integration step as a differentiable, GPU-native component, rather than precomputing a frozen magnitude file with synphot, trades a real implementation cost (reimplementing SED × bandpass integration from scratch in autodiff-native form, rather than reusing synphot's codebase) for keeping spectral-response co-design in scope. The alternative — precomputed magnitude files, bandpass fixed from the model's point of view — is simpler now but forecloses that capability permanently; regenerating files for a new bandpass is an offline sweep, not something a gradient can drive.

synphot alone is the wrong container for cube-shaped R&D validation data — its native `SourceSpectrum` is strictly 1D. specutils' `Spectrum`/`Spectrum1D` fills that gap without displacing synphot's role as the photometric-integral engine, since `Observation` already accepts it as an input type. The remaining open question (batched vs. per-spaxel evaluation) affects R&D-phase convenience only; it has no bearing on the eventual differentiable component, which is a from-scratch vectorized implementation regardless of how synphot behaves.

## Consequences

- **Easier:** correct band-integrated magnitudes, AB/SWIR support, physically grounded backgrounds (zodiacal, Earth) — none of which SatSim will ever natively provide.
- **Easier:** each pipeline stage can be developed, tested, and understood independently (this is also why the SatSim tutorial notebook is being built first, in isolation).
- **Harder:** all inter-tool registration is our responsibility — hypercube/spectral consistency between DIRSIG and synphot, target-injection and light-curve mechanism design, PSF pre-blur on injected backgrounds, background-ephemeris/time indexing, and CZML export/import verification.
- **Harder:** the differentiable reimplementation of the band-integration step must keep intermediate quantities in linear flux/count space, not magnitude. Magnitude is logarithmic and non-additive, and $d(\text{mag})/d(\text{flux}) \propto 1/\text{flux}$ diverges as flux→0 — exactly the faint-source regime this work cares about. Magnitude should remain the human-facing/QC output, never an internal node the graph differentiates through.
- **To revisit:** which autodiff framework (JAX, PyTorch, or TensorFlow) implements the refactored SatSim and the differentiable band-integration component (currently prototyped as synphot, to be retired once the reimplementation exists) — undecided; the ADR intentionally does not presume an answer. The RSO-injection/light-curve mechanism itself is undesigned (TBD) — needs a decision on what target properties vary (brightness only, or attitude/BRDF-driven), and how that interacts with the differentiable band-integration layer versus SatSim's own rendering step. SatViz/Cesium's actual CZML-loading interface (unconfirmed); validity of SatSim's built-in sky-background presets for non-visible bands; whether synphot's `Observation` batch-integrates a multidimensional specutils `Spectrum1D` or requires per-spaxel looping.

## Action Items

1. [ ] Build "Using SatSim" tutorial notebook — SatSim only, no synphot/DIRSIG/SatViz, following SatSim's readthedocs usage example with liberal markdown commentary.
2. [ ] Determine visualization approach from the notebook's output — verify SatSim's CZML export contents and whether SatViz/`satsimjs` exposes a CZML loader, with Skyfield/poliastro as fallback.
3. [ ] R&D validation: prototype the band-integration physics using synphot against a DIRSIG-shaped test cube held in a specutils `Spectrum`/`Spectrum1D` container. Confirm with a small toy cube whether `synphot.Observation` batch-integrates across the multidimensional flux array in one call, or requires per-spaxel iteration — do not assume either way.
4. [ ] Design and implement a new differentiable, GPU-native band-integration component (linear flux/count space internally), operating on DIRSIG hypercubes, reproducing the physics validated in step 3. This is a reimplementation, not a refactor of synphot's codebase — synphot is retired once this component exists. Framework (JAX/PyTorch/TensorFlow) to be decided as part of this task, not assumed in advance.
5. [ ] Audit SatSim's actual source (not just its published architecture) for gradient-breaking operations, to confirm Poisson noise sampling and annotation/truth-file logic are the only two exceptions. This assumption is currently inferred from TensorFlow's known autodiff coverage, not verified line-by-line against SatSim's codebase.
6. [ ] (Later) Patch SatSim's non-differentiable exceptions confirmed by the audit above: replace Poisson noise sampling with a differentiable Gaussian reparameterization (valid at operating photon-count regimes), and exclude annotation/truth-file logic from the differentiable graph (it never needs a gradient). This is the concrete content of the BLUF's "exceptions to be addressed."
7. [ ] (Later) DIRSIG background precompute and injection — noiseless, PSF-pre-blurred, ephemeris-indexed.
8. [ ] (Later) Design the RSO-injection/light-curve mechanism — target-property variation over a frame sequence, interface with SatSim's rendering step.
9. [ ] (Later) Confirm and implement SatViz/Cesium integration against verified source behavior.
10. [ ] Standing instruction for Claude Code: prefer vectorized/GPU-compatible-adjacent implementations when cost-neutral outside the band-integration component; never force this at the expense of correctness or exploration speed during R&D. The eventual differentiable reimplementation of synphot's functionality is held to the differentiable/GPU-native requirement as a firm decision, not a preference — synphot itself is R&D scaffolding and is not the thing being made differentiable.
