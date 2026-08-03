# CLAUDE.md

## Project Phase

R&D exploration toward an eventual differentiable RSO-simulation model. Current task: "Using SatSim" tutorial notebook — SatSim only, no synphot/DIRSIG/SatViz. See `docs/adr/0001-hybrid-synphot-satsim-pipeline.md` for full architecture and rationale; do not restate its reasoning here.

Action items in the ADR are sequenced. Steps described in the ADR as "firm decisions" (e.g., the differentiable band-integration reimplementation, patching SatSim's noise/annotation operations) are still gated behind earlier steps and must not be started unprompted, even though the ADR states them as decided rather than optional.

## Library Defaults

- Default to the astropy ecosystem for modeling, radiometric conversions, physical constants, and unit-aware calculations, unless a specific pipeline tool (SatSim, DIRSIG) requires otherwise.
- When using the astropy ecosystem, use `astropy.units`/`astropy.constants` throughout and enforce units at every function/module boundary — no bare floats crossing an interface where a physical quantity is implied.

## Working from `prompt.md`

Complex instructions are authored in claude.ai and saved as `prompt.md` at repo root. This is copied into a Claude Code session with a standard instruction to execute it and report results.

## Environment

This project uses an existing conda environment, `satsim-env` (Python 3.10, conda-forge, ipykernel-registered). Do not create a new venv or conda environment. Activate `satsim-env` for all Python execution and package installs (e.g., `conda run -n satsim-env python ...` or `conda activate satsim-env && ...`). For any Jupyter notebook, set the kernel to the `satsim-env` kernelspec rather than defaulting to base or creating a new one.

### On Giving Instructions to Claude Code

- Be explicit and literal. State each step as its own direct instruction: "implement X from `prompt.md`, then stop and report back."
- One task at a time. Never implement beyond the stated scope.
- **Never call `ExitPlanMode` or `AskUserQuestion` when told to "produce a plan, then stop."** Write the plan as plain text and end the turn. These tools are one accidental click from executing an unreviewed plan. Only call `ExitPlanMode` when explicitly told to implement.
- **Planning mode workflow:** "enter planning mode and produce a plan from `prompt.md`, then stop." Review in chat. Once approved: "implement the approved plan, then stop and report back." Do not write a separate implementation prompt that overrides the approved plan.
- **`prompt.md` convention:** all Claude Code instructions go in `prompt.md` at repo root. Short inline chat instructions direct Claude Code to read that file.
- **`/run` is not an implementation skill.** Run-and-verify only.
