# Context Transfer: synphot → SatSim → SatViz/Cesium Implementation

**Objective.** Implement a working pipeline: synthetic photometry (synphot) generates per-band magnitudes from SEDs/bandpasses/QE → SatSim renders labeled RSO detection frames using those magnitudes plus injected backgrounds → SatViz/Cesium visualizes the resulting scene geometry. This thread is for code; prior thread covered architecture and tool evaluation only — no code was written.

**Role calibration for this thread.** Kevin has no hands-on coding background — engages as physicist/system architect. Expect Claude to write and iterate code directly; Kevin reviews, redirects, and catches conceptual errors rather than debugging syntax.

## Pipeline stages and division of labor

**synphot (astropy-affiliated) — band integration, the piece SatSim can't do.** `SpectralElement` (bandpass/QE/filter throughput) × `SourceSpectrum` (SED) → `Observation` → magnitude. Supports AB magnitudes natively; not limited to Vega. This is the fix for SatSim's missing color term — compute per-band, per-source magnitude here, hand the result to SatSim as a literal magnitude.

**SatSim — image formation engine, not a radiometric model.** Pure Python/TensorFlow, GPU-accelerated, config via plain dict/JSON (`image_generator`/`gen_multi` in `satsim.satsim`). No schema validation — bad config values fail silently rather than raising, so test configs carefully. Brightness is always a scalar: literal magnitude, or a Lambertian-sphere model (albedo, size, distance, phase angle) computed internally from sun/moon geometry. No wavelength axis anywhere — `fpa.zeropoint` absorbs the entire $\int T(\lambda)\eta(\lambda)\,d\lambda$ integral as one number.

**SatViz/Cesium — visualization, still partially unresolved.** `satsimjs` is a Cesium-based 3D scene library (`Universe` + `createViewer`), with an optional client-server runtime (`RuntimeClient`/`SessionManager`/`SimulationRuntime`/`HttpRuntimeServer`) for live, multi-client scenario control. `satviz` (pip) is a thin Jupyter/Marimo anywidget wrapping the same Cesium viewer for notebook embedding — does not wrap the runtime layer. **Confirmed bridge:** SatSim has native CZML output (`save_czml` option, includes sensor visualization and object billboard). CZML is Cesium's own time-dynamic scene format, so this is the legitimate path to feed SatSim geometry into the Cesium viewer. **Not confirmed:** whether `satsimjs`/`satviz`'s `Universe` API has a documented CZML-loading convenience method, or whether raw Cesium `CzmlDataSource.load()` needs to be called against the underlying viewer object via the widget's JS. Treat any specific claim about this (from any source, including other LLMs) as unverified until checked against `satsimjs` source directly — a prior claim about a "direct JSON injection" API parsing `obs`/`tracker`/`gimbal` blocks did not hold up against the actual README and was very likely confabulated.

**Fallback geometry-check path, no new dependency risk.** Skyfield (`EarthSatellite` from TLE, already in toolchain) + poliastro/matplotlib 3D plot, reading SatSim's own JSON config (`geometry.site`, `geometry.site.track.tle1/tle2`) programmatically — no transcription, guaranteed to work, just not a live Cesium globe.

## Established design decisions (apply these, don't re-derive)

**One brightness authority per source, never two.** Either feed SatSim a finished magnitude (synphot output — preferred) or let SatSim's Lambertian-sphere model compute distance/phase falloff internally. Mixing both double-counts $1/r^2$ and phase.

**PSF must be applied to any injected background before it reaches SatSim.** SatSim convolves its own point-source layer with the PSF kernel but never touches externally-injected background arrays (zodiacal, DIRSIG Earth-background). Convolve upstream (in DIRSIG, or manually) or the background renders unphysically sharp next to a properly-blurred point-source target.

**Keep any injected background (e.g. DIRSIG) strictly linear/noiseless.** Perfect noiseless detector, PSF applied, no gain nonlinearity or well-clipping — this is what lets SatSim's own shot-noise draw be the only noise source (avoiding double-counting noise) and lets the same precomputed background be rescaled for different exposure times by simple multiplication.

**Precompute-and-amortize is the standing pattern.** Render the expensive background (DIRSIG orbit/ground-track strip) once; run cheap SatSim Monte Carlo (target trajectory, brightness, detection algorithm variations) many times against it. Same principle as the IF-AXS lookup-table and DIRSIG_URSO cookbook architectures elsewhere in this project. Requires a background ephemeris/index (time or orbital-position → stored frame/strip lookup) decoupled from the target's own clock.

**This pipeline is not differentiable, and won't become so.** DIRSIG and SatSim stay black-box forward models. Any future differentiable capability is a separate surrogate trained on this pipeline's output — consistent with the existing JAX (forward/Fisher) vs. PyTorch (detection/deployment) simulate-then-train boundary.

**Ground-based vs. space-based, RPO vs. absolute propagation.** SatSim's ground-based mode (topocentric site, sidereal/rate track, POPPY PSF with turbulence, sky background models) is mature; space-based observer mode is newer/less-exercised. For range/angular-rate-relative-to-platform scenarios, use SatSim's RPO generator, not TLE/SGP4 absolute propagation.

**Band-agnostic core cuts both ways for AB/SWIR.** No native AB or SWIR support, but nothing prevents either — SatSim's scalar zeropoint doesn't care what system or band produced it. Star catalogs (SSTR7/Hipparcos/Gaia) are Vega/optical-band only; any SWIR or AB-system star field needs magnitudes computed via synphot, not pulled from catalog defaults. Same caution applies to built-in sky-background presets (skyglow/moonlight/twilight) — likely visible-band-calibrated, verify before reusing for SWIR.

## Actual next steps, in order

**1. "Using SatSim" Jupyter notebook — personal tutorial/reference, built first.** Will likely follow the example in SatSim's readthedocs usage docs, with liberal markdown commentary. Deliberately excludes synphot, DIRSIG, and SatViz at this stage — pure SatSim exploration: config structure, `image_generator`/`gen_multi`, site/track/TLE geometry, star catalogs, basic RSO injection, rendered output inspection.

**2. Visualization exploration.** Once the notebook produces frames and geometry, determine what/how to visualize — this is where the SatViz/CZML question (and the Skyfield/poliastro fallback) gets resolved concretely rather than architecturally.

**3. Incorporate synphot.** Add band-integrated magnitude computation once the SatSim baseline and visualization approach are both working.

**4. DIRSIG background injection and SatViz/Cesium integration** follow later, per the architecture above — not part of the initial build sequence.
