You are continuing R&D work on the RSO simulation pipeline (SatSim/synphot/DIRSIG
hybrid image chain). Read docs/adr/0001-hybrid-synphot-satsim-pipeline.md and
CLAUDE.md before doing anything else. Also read docs/adr/0002-notebook-sequence-
and-pydantic-timing.md — it defines the notebook sequence and where pydantic
enters.

I am stepping away for several hours. Work through the queue below unattended.
Do not wait for input on anything except the items explicitly marked HARD STOP.

ORDER OF WORK

0. Environment hygiene (do this first, it's overdue): pin the satsim-env
   dependencies you had to install ad hoc while building the tutorial notebook
   (scikit-image, opensimplex, poppy, apng, imagecodecs, czmlpy, asdf, pooch,
   Click, Cython, pygc, nbformat, nbconvert) into a requirements.txt or
   environment.yml. Verify a clean env can be rebuilt from it if that's cheap
   to check; otherwise just produce the pinned file.

1. Notebook: visualization approach (ADR Action Item 2). Verify what SatSim's
   save_czml actually exports. Determine whether SatViz/satsimjs exposes a
   CZML loader you can drive from here, or whether you need the Skyfield/
   poliastro fallback. Build the notebook, get it running top-to-bottom, no
   synphot/DIRSIG in scope. Report what you found against what the ADR
   assumed.

2. Notebook: band-integration validation (ADR Action Item 3). Build a toy
   DIRSIG-shaped hypercube in a specutils Spectrum/Spectrum1D container —
   synthetic data only, do not invoke DIRSIG itself for this (see HARD STOPS).
   Determine empirically whether synphot.Observation batch-integrates across
   the multidimensional flux array in one call or requires per-spaxel
   iteration — don't assume either way, show the test. Validate the SED x
   bandpass x QE -> magnitude physics against a hand-computed reference case.

3. Differentiability audit (ADR Action Item 5) — READ-ONLY, do not patch
   anything. Read SatSim's actual installed source (not just its docs) and
   confirm or refute that Poisson noise sampling and annotation/truth-file
   logic are the only two gradient-breaking operations in its rendering path.
   Write up the audit findings with file/line references. Do not attempt
   Action Item 6 (patching) — that's gated behind this audit being reviewed
   by me and behind the framework decision in item 4 below.

4. Injection-mechanism reconnaissance only (feeds ADR Action Item 8, pulled
   forward) — investigate, do not design or implement. Survey SatSim's
   rendering API for where/how an externally-supplied background image could
   be injected before its noise draw. Identify the actual hook points. Write
   up the open design questions this surfaces (e.g. what should vary across a
   frame sequence — brightness only, or attitude/BRDF-driven glint — and how
   that interacts with the band-integration layer). Do not build an injection
   mechanism or pick an approach.

HARD STOPS — do not decide or act on these, surface them in the report instead

- Autodiff framework choice (JAX vs PyTorch vs TensorFlow) — ADR Action Item
  4 explicitly defers this to a dedicated decision. Do not start the
  differentiable band-integration component regardless of how much time is
  left.
- The RSO-injection/light-curve mechanism design itself — investigate per
  item 4 above, but the actual design choice is mine to make.
- DIRSIG. It is licensed software already installed on this machine. You must
  not install, download, reconfigure, license, or otherwise attempt to acquire
  or modify it under any circumstances, regardless of what happens in this
  session — this is not something you have the ability or the standing to do.
  It is also simply out of scope tonight: ADR Action Item 7 (DIRSIG background
  precompute/injection) is sequenced "Later," behind the framework decision
  and the injection-mechanism design, so do not invoke DIRSIG even though it
  is present and usable on this machine.
- Any RSO_Sim package/class scaffolding or pydantic introduction — per the
  roadmap doc, both wait until the injection-mechanism design in item 4 is
  actually settled by me. Investigation notebooks only, no library code yet.

OPERATING RULES

- Work on a new branch (e.g. rnd/action-items-2-5), not main. Commit after
  each numbered item completes, with a clear message. Do not push anywhere.
- If you hit a genuinely ambiguous call that isn't one of the hard stops
  above, make the most reasonable choice, state the assumption plainly in
  the final report, and keep going — don't block waiting for me.
- If the same failure repeats more than ~3 times on one item, stop that item,
  log it as blocked with what you tried, and move to the next one rather than
  looping.
- SatSim has no schema validation — verify results by inspecting actual
  output (rendered frames, exported files), not just "ran without error."
- When you finish, or when you run out of non-hard-stop work, write a single
  status report (append to docs/adr/status.md or a new dated file) covering:
  what was completed, what's blocked and why, all assumptions you made, and
  the open decisions now waiting on me (framework choice, injection design,
  anything else that came up).
