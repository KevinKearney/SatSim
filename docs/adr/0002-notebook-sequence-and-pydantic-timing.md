# ADR-0002: Notebook sequence and pydantic timing for RSO_Sim

**Status:** Proposed
**Date:** 2026-08-03
**Deciders:** Kevin

Companion to ADR-0001. Captures sequencing decisions made before Notebooks 2-5 exist.

## Naming/scope

`RSO_Sim` is the working name for the eventual integration package spanning synphot, SatSim, and DIRSIG. The name is reserved but the package has no defined API yet -- synphot/SatSim/DIRSIG remain independent per ADR-0001 through at least Action Items 2-3. Do not scaffold `RSO_Sim` modules/classes before Notebook 4 (see below); doing so earlier means designing to an interface that hasn't been observed yet.

## Notebook sequence (mapped to ADR-0001 Action Items)

1. **Using SatSim** (Action Item 1) -- complete. SatSim in isolation, no abstraction imposed.
2. **Geometry/visualization** (Action Item 2) -- CZML export contents, SatViz/Skyfield viability. Independent of the radiometric chain.
3. **Band-integration validation** (Action Item 3) -- synphot + specutils `Spectrum1D` against a toy DIRSIG-shaped cube; resolves batched-vs-per-spaxel integration.
4. **Injection/light-curve prototype** (pulls Action Item 8 forward) -- first point SatSim and synphot/DIRSIG output must interoperate (magnitude handoff, PSF pre-blur, time-varying brightness). **This is the first point class extraction is justified** -- it's the first repeated cross-tool config/data handoff, i.e. the first real class boundary (rule of three: need >=2 tools' interfaces meeting before generalizing).
5. **Differentiability audit** (Action Item 5) -- source-level check of SatSim's graph for the Poisson-sampling and annotation-logic exceptions.

Rule: do not extract classes off Notebooks 1-3 individually. Each exercises one tool in isolation; the class boundary only becomes visible once two tools' config/data actually have to reconcile, which first happens at Notebook 4.

## Pydantic -- when and how

**When:** introduce at Notebook 4, not before. Rationale: pydantic's role here is to be the schema-validation layer SatSim structurally lacks (see status.md's silent-failure catalogue -- `spatal_osf` typo, `magnitude`/`mv` key confusion, `stars.mv.bins`/`density` N+1-vs-N mismatch). That wrapper is only meaningful once RSO_Sim owns a config boundary between tools, which doesn't exist until Notebook 4.

**Scope constraint -- config only, never tensors:** pydantic models describe *what to simulate* (site, target, FPA, spectral bandpass params) -- validated, immutable, human/config-facing. They must never wrap the DIRSIG hypercube or SatSim's rendered arrays; that data stays plain numpy/TF/JAX so it can later sit inside an autodiff graph. This preserves the same config/data boundary the ADR draws around magnitude-vs-linear-flux (magnitude is QC/human-facing, never a differentiated node) -- pydantic stays strictly on the QC/config side of that same line.

**Implementation constraint -- adapter, not patch:** the pydantic layer should wrap SatSim's/DIRSIG's raw dict/array formats via an explicit `to_satsim_config()`-style export method. It is RSO_Sim's own validated front door, not a fork or patch of SatSim's config handling -- consistent with the ADR's general stance of not refactoring external tools.

**Units:** pydantic does not natively validate physical units or array shapes -- a `float` type check passes regardless of unit correctness. Given the ADR's emphasis on linear-flux vs. magnitude correctness, pair pydantic field validators with `astropy.units` (or `pint`) for magnitude/wavelength/flux fields at the same Notebook 4 juncture, rather than treating unit-correctness as a separate future decision.
